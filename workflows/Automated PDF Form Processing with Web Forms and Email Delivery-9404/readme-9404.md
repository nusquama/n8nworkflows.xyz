Automated PDF Form Processing with Web Forms and Email Delivery

https://n8nworkflows.xyz/workflows/automated-pdf-form-processing-with-web-forms-and-email-delivery-9404


# Automated PDF Form Processing with Web Forms and Email Delivery

### 1. Workflow Overview

This workflow automates the process of collecting user inputs via a web landing page form, filling a PDF form with this data, and then emailing the completed PDF form to a predefined email address. It is designed for use cases such as insurance quote requests or any scenario requiring user information submission followed by PDF form generation and email delivery.

The workflow is logically organized into the following blocks:

- **1.1 Landing Page Delivery**: Hosts and serves a custom HTML landing page with a form for users to submit their personal details.
- **1.2 Form Data Reception**: Receives form submissions from the landing page and triggers the PDF processing.
- **1.3 PDF Retrieval and Filling**: Downloads a template PDF form, fills it dynamically with the submitted data.
- **1.4 Optional Form Field Inspection**: (Optional) Extracts and lists PDF form field names for reference.
- **1.5 Email Sending**: Sends the filled PDF as an attachment via SMTP email.

---

### 2. Block-by-Block Analysis

#### 1.1 Landing Page Delivery

- **Overview:**  
  This block serves a custom HTML landing page that includes a form for users to fill with their personal details. It dynamically injects the form submission endpoint URL into the page to ensure the form posts data back correctly to the workflow.

- **Nodes Involved:**  
  - Landingpage Endpoint (Webhook)  
  - Set Form Endpoint  
  - HTML for Landingpage  
  - Respond to Webhook  
  - Sticky Note1 (Documentation)

- **Node Details:**

  - **Landingpage Endpoint**  
    - *Type:* Webhook  
    - *Role:* Entry point serving the landing page HTML. Listens for HTTP GET requests at a specific path.  
    - *Configuration:* Path set uniquely to expose the landing page; response mode configured to use a downstream "Respond to Webhook" node to deliver HTML content.  
    - *Connections:* Outputs to "Set Form Endpoint".

  - **Set Form Endpoint**  
    - *Type:* Set  
    - *Role:* Injects the dynamic webhook URL (the form data submission endpoint) into the JSON data as `FormEndpoint`.  
    - *Configuration:* Assigns `FormEndpoint` with the webhook URL from the incoming request. This is used inside the HTML to configure the form’s submission target.  
    - *Connections:* Outputs to "HTML for Landingpage".

  - **HTML for Landingpage**  
    - *Type:* HTML  
    - *Role:* Contains the full HTML content of the landing page including Tailwind CSS styling, a multi-field form, success/error handling, and JavaScript to submit the form data asynchronously to the specified `FormEndpoint`.  
    - *Key Expressions:* Uses `{{ $json.FormEndpoint }}` to dynamically set the form's POST URL.  
    - *Connections:* Outputs to "Respond to Webhook".

  - **Respond to Webhook**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends the HTML content from the previous node as the HTTP response to the landing page request.  
    - *Configuration:* Responds with the `html` field from JSON, content type is text/html implicitly.  

  - **Sticky Note1**  
    - *Content:* "## Landingpage Server\nYou can access the landing page at the webhook address."  
    - *Purpose:* Documentation for users informing them how to access the landing page.

- **Edge Cases / Potential Failures:**  
  - If the webhook URL changes or is incorrectly injected, the form submission will fail.  
  - Network or permission issues serving the landing page HTTP GET request.

---

#### 1.2 Form Data Reception

- **Overview:**  
  This block receives the POSTed form data from the landing page. It acts as a webhook endpoint to accept JSON payloads containing user input, then triggers the PDF processing.

- **Nodes Involved:**  
  - FormData Endpoint (Webhook)  
  - HTTP Request  
  - Sticky Note2 (Documentation)

- **Node Details:**

  - **FormData Endpoint**  
    - *Type:* Webhook  
    - *Role:* Receives form data via HTTP POST at a defined path.  
    - *Configuration:* Path matches the URL injected into the landing page form's action attribute. Response data is a simple HTML message `<h1>Hello World</h1>` for acknowledgment.  
    - *Connections:* Outputs to "HTTP Request".

  - **HTTP Request**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the template PDF form from a fixed URL.  
    - *Configuration:* URL points to a publicly accessible PDF form template file. No additional options set.  
    - *Connections:* Outputs to "PDF Form Fill (Fill PDF Fields)" and "Get PDF Form Fields".

  - **Sticky Note2**  
    - *Content:* "## FormData Endpoint\nThe data for the form is received here from the landing page."  
    - *Purpose:* Clarifies the webhook role for receiving data.

- **Edge Cases / Potential Failures:**  
  - Timeout or failure downloading the PDF template (HTTP Request).  
  - Invalid or missing form data in the POST request.  
  - Webhook path mismatch causing form submission failure.

---

#### 1.3 PDF Retrieval and Filling

- **Overview:**  
  This block processes the downloaded PDF and fills its form fields with the user-submitted data. The filled PDF is then prepared for email attachment.

- **Nodes Involved:**  
  - PDF Form Fill (Fill PDF Fields)  
  - Get PDF Form Fields  
  - Sticky Note (Optional documentation)

- **Node Details:**

  - **PDF Form Fill (Fill PDF Fields)**  
    - *Type:* Custom PDF Toolkit node (PdfFormFill)  
    - *Role:* Takes the downloaded PDF and fills specified form fields with values extracted from the JSON body of the form submission.  
    - *Configuration:* Maps form fields by name to values such as `firstName`, `lastName`, `address`, `houseNo`, `postalCode`, `city`, `country`. Also sets a checkbox field "Language 1 Check Box" to checked ("X").  
    - *Credentials:* Uses a custom JavaScript API credential.  
    - *Key Expressions:* Field values use expressions like `={{ $json.body.firstName }}` to reference form submission data.  
    - *Connections:* Outputs to "Send email".

  - **Get PDF Form Fields**  
    - *Type:* Custom PDF Toolkit node (GetFormFieldNames)  
    - *Role:* Optionally extracts and lists all form field names in the PDF for reference or debugging.  
    - *Credentials:* Uses a separate custom JavaScript API credential.  
    - *Connections:* Not connected downstream; serves informational purpose.

  - **Sticky Note**  
    - *Content:* "## Form Field Names (Optional)\nYou can use this module to read the names of the form fields."  
    - *Purpose:* Indicates that the "Get PDF Form Fields" node is optional and used to inspect PDF field names.

- **Edge Cases / Potential Failures:**  
  - PDF form field names mismatch or missing fields cause incomplete or failed filling.  
  - Custom JS API credential failure or rate limits.  
  - Incorrect data types or missing data in form submission JSON.  
  - PDF processing timeouts or errors.

---

#### 1.4 Email Sending

- **Overview:**  
  Sends the filled PDF form as an email attachment to a predetermined recipient email address.

- **Nodes Involved:**  
  - Send email

- **Node Details:**

  - **Send email**  
    - *Type:* Email Send  
    - *Role:* Sends email with the filled PDF attached.  
    - *Configuration:*  
      - Subject: "Filled PDF Form"  
      - To: "test@testmail.com" (hardcoded recipient)  
      - From: "test@test.de" (hardcoded sender)  
      - Body: Plain text with a polite message and signature.  
      - Attachments: Uses the filled PDF binary data output from previous node.  
    - *Credentials:* Uses SMTP credential for authentication.  
    - *Webhook ID:* Set, indicating this node is part of a webhook triggered flow.  
    - *Connections:* None downstream (end node).

- **Edge Cases / Potential Failures:**  
  - SMTP server authentication or connectivity issues.  
  - Attachment size limits or encoding errors.  
  - Incorrect email addresses causing delivery failure.  
  - Rate limits or spam filter blocking.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                     | Input Node(s)                 | Output Node(s)                       | Sticky Note                                                                                   |
|-----------------------------|-----------------------------------|-----------------------------------|------------------------------|------------------------------------|----------------------------------------------------------------------------------------------|
| Landingpage Endpoint         | Webhook                           | Serves landing page HTML           |                              | Set Form Endpoint                  | ## Landingpage Server You can access the landing page at the webhook address.                 |
| Set Form Endpoint            | Set                               | Injects form submission endpoint   | Landingpage Endpoint          | HTML for Landingpage               |                                                                                              |
| HTML for Landingpage         | HTML                              | Contains full landing page content | Set Form Endpoint             | Respond to Webhook                 |                                                                                              |
| Respond to Webhook           | Respond to Webhook                 | Sends HTML response                 | HTML for Landingpage          |                                    |                                                                                              |
| FormData Endpoint            | Webhook                           | Receives form submission data      |                              | HTTP Request                     | ## FormData Endpoint The data for the form is received here from the landing page.           |
| HTTP Request                | HTTP Request                      | Downloads PDF template              | FormData Endpoint             | PDF Form Fill (Fill PDF Fields), Get PDF Form Fields |                                                                                              |
| PDF Form Fill (Fill PDF Fields) | Custom PDF Toolkit (PdfFormFill) | Fills PDF form with user data       | HTTP Request                 | Send email                       | ## Form Field Names (Optional) You can use this module to read the names of the form fields. |
| Get PDF Form Fields          | Custom PDF Toolkit (GetFormFieldNames) | Extracts PDF form field names       | HTTP Request                 |                                    | ## Form Field Names (Optional) You can use this module to read the names of the form fields. |
| Send email                  | Email Send                       | Sends filled PDF via email          | PDF Form Fill (Fill PDF Fields) |                                    |                                                                                              |
| Sticky Note1                | Sticky Note                      | Documentation                      |                              |                                    | ## Landingpage Server You can access the landing page at the webhook address.                 |
| Sticky Note2                | Sticky Note                      | Documentation                      |                              |                                    | ## FormData Endpoint The data for the form is received here from the landing page.           |
| Sticky Note                 | Sticky Note                      | Documentation                      |                              |                                    | ## Form Field Names (Optional) You can use this module to read the names of the form fields. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Landingpage Endpoint"**  
   - Type: Webhook  
   - HTTP Method: GET  
   - Path: Unique path (e.g., "db534064-7845-44c2-abdd-3f2189772a27")  
   - Response Mode: "Response Node" (to connect downstream response)  

2. **Create Set Node "Set Form Endpoint"**  
   - Assign a new string field named `FormEndpoint`  
   - Set value to `{{ $json.webhookUrl }}` (the webhook URL of the form data endpoint, to be created next)  
   - Connect "Landingpage Endpoint" output to this node.

3. **Create Webhook Node "FormData Endpoint"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique path (can be same as above or another, used in form submission)  
   - Response Data: `<h1>Hello World</h1>` or any acknowledgment HTML message  
   - Connect this node output next to "HTTP Request".

4. **Update "Set Form Endpoint" node's `FormEndpoint` to the webhook URL of the "FormData Endpoint" node**  
   - This URL will be injected into the landing page HTML for form submission.

5. **Create HTML Node "HTML for Landingpage"**  
   - Paste the full HTML content of the landing page (provided above)  
   - Replace the form submission URL dynamically using `{{ $json.FormEndpoint }}` expression in the JavaScript fetch call URL.  
   - Connect "Set Form Endpoint" output to this node.

6. **Create Respond to Webhook Node "Respond to Webhook"**  
   - Respond with `"text"` content type  
   - Set response body to `{{ $json.html }}` from previous node  
   - Connect "HTML for Landingpage" output to this node.

7. **Connect "Landingpage Endpoint" → "Set Form Endpoint" → "HTML for Landingpage" → "Respond to Webhook"**

8. **Create HTTP Request Node "HTTP Request"**  
   - Method: GET  
   - URL: `https://community.adobe.com/havfw69955/attachments/havfw69955/acrobat-reader/109709/1/EDIT%20OoPdfFormExample.pdf` (template PDF form)  
   - Connect "FormData Endpoint" output to this node.

9. **Create PDF Form Fill Node "PDF Form Fill (Fill PDF Fields)"**  
   - Use the Custom PDF Toolkit node for filling PDF forms  
   - Map form fields names to expressions referencing incoming JSON body properties (e.g., `{{ $json.body.firstName }}`, `{{ $json.body.lastName }}`, etc.)  
   - Set checkbox field "Language 1 Check Box" to "X" to mark checked  
   - Assign appropriate CustomJS API credential  
   - Connect "HTTP Request" output to this node.

10. **Create Send Email Node "Send email"**  
    - Use SMTP credentials configured with your mail server  
    - To Email: "test@testmail.com" (replace with target recipient)  
    - From Email: "test@test.de" (replace with sender address)  
    - Subject: "Filled PDF Form"  
    - Body: Plain text with a short message  
    - Attachments: Attach the filled PDF binary data from previous node  
    - Connect "PDF Form Fill (Fill PDF Fields)" output to this node.

11. *(Optional)* Create node "Get PDF Form Fields"  
    - Use Custom PDF Toolkit node to extract form field names  
    - Assign separate CustomJS API credential  
    - Connect "HTTP Request" output to this node  
    - This node is for inspecting PDF field names and is not connected downstream.

12. **Verify all credentials:**  
    - CustomJS API credentials for PDF nodes  
    - SMTP credentials for email node  

13. **Test the workflow:**  
    - Access the landing page via the "Landingpage Endpoint" webhook URL  
    - Submit the form with test data  
    - Verify email reception with the filled PDF attached

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The landing page uses Tailwind CSS for styling and includes JavaScript to handle form submission asynchronously.       | Embedded in the HTML for Landingpage node                      |
| The form submission endpoint URL is dynamically injected into the landing page to ensure accurate POST target.         | Set Form Endpoint node usage                                   |
| PDF template is publicly hosted on Adobe Community forums for demonstration purposes.                                   | PDF URL in HTTP Request node                                   |
| Custom PDF Toolkit nodes require specific CustomJS API credentials, which must be configured in n8n credentials panel. | PDF Form Fill and Get PDF Form Fields nodes                    |
| Email sending requires SMTP credentials configured with valid server and authentication details.                       | Send email node                                                |
| Sticky notes provide contextual documentation directly in the workflow editor for clarity.                            | See Sticky Note, Sticky Note1, Sticky Note2                    |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. All processing complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.