Generate Verifiable CEU Certificates with Google Workspace & QR Verification

https://n8nworkflows.xyz/workflows/generate-verifiable-ceu-certificates-with-google-workspace---qr-verification-11012


# Generate Verifiable CEU Certificates with Google Workspace & QR Verification

### 1. Workflow Overview

This workflow automates the generation and delivery of verifiable Continuing Education Unit (CEU) certificates for corporate training events using Google Workspace and other integrated services. It accepts certificate request data via a webhook, validates input and email authenticity, generates a unique certificate ID and QR code, creates a professional PDF certificate, uploads it to Google Drive, emails it to the recipient, notifies organizers via Slack, logs the event in Google Sheets, and finally responds to the original webhook request with certificate details.

**Target Use Cases:**  
- Corporate training providers issuing CEU certificates  
- Educational institutions delivering verifiable training certificates  
- Automations requiring secure, audit-trailed certificate generation with email and Slack notifications  

**Logical Blocks:**

- **1.1 Input Reception & Validation:** Receive webhook POST requests with certificate data; validate required fields and email authenticity.  
- **1.2 Certificate ID & QR Code Generation:** Generate a unique certificate ID and corresponding verification QR code.  
- **1.3 Certificate Rendering & PDF Generation:** Render the certificate as an HTML document and convert it to a PDF file.  
- **1.4 File Handling & Storage:** Download the generated PDF, upload it to Google Drive.  
- **1.5 Notification & Logging:** Send the certificate via email; notify organizers on Slack; log certificate data in Google Sheets.  
- **1.6 Final Response:** Return success or error response to the original webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

**Overview:**  
This block receives incoming certificate requests via webhook, verifies all required fields are present, and validates the recipient’s email address to ensure it is deliverable and not disposable.

**Nodes Involved:**  
- Webhook Trigger  
- Validate Required Fields (If node)  
- Stop: Incomplete Data (Respond to Webhook)  
- Verifi Email  
- Check Domain Verified (If node)  
- Stop: Domain Not Verified (Respond to Webhook)

**Node Details:**

- **Webhook Trigger**  
  - Type: Webhook Trigger  
  - Role: Entry point accepting POST requests at path `/ceu-certificate`.  
  - Configuration: HTTP POST; response mode set to use a response node (deferred response).  
  - Inputs: External HTTP request with JSON body containing certificate data fields.  
  - Outputs: Passes request JSON to validation node.  
  - Error Cases: Missing or malformed POST data, timeout if no data received.

- **Validate Required Fields**  
  - Type: If node  
  - Role: Checks that `name`, `email`, `courseTitle`, `CEUsEarned`, and `completionDate` fields are all present and non-empty in the incoming JSON.  
  - Configuration: All conditions combined with string `isNotEmpty` checks on body fields.  
  - Success path if all fields present; failure leads to error response.  
  - Edge Cases: Missing fields cause workflow to stop with error response.

- **Stop: Incomplete Data**  
  - Type: Respond to Webhook  
  - Role: Returns HTTP 400 error JSON response indicating missing required fields.  
  - Configuration: Response code 400; JSON with success=false and relevant error message.  
  - Inputs: Triggered if required fields validation fails.

- **Verifi Email**  
  - Type: VerifiEmail node  
  - Role: Validates the provided email address for deliverability, checking MX records and disposable domain status.  
  - Configuration: Uses the email from incoming JSON.  
  - Credentials: Requires VerifiEmail API credentials.  
  - Outputs: JSON with `valid` boolean field indicating email legitimacy.  
  - Failure Modes: API auth error, network errors, invalid email format.

- **Check Domain Verified**  
  - Type: If node  
  - Role: Branches workflow based on `valid` boolean output from Verifi Email node.  
  - Configuration: True if valid; false otherwise.  
  - False path triggers domain not verified stop node.

- **Stop: Domain Not Verified**  
  - Type: Respond to Webhook  
  - Role: Returns HTTP 400 error JSON response indicating invalid or disposable email.  
  - Configuration: Response code 400; JSON with success=false and error message.  
  - Inputs: Triggered if email verification fails.

---

#### 1.2 Certificate ID & QR Code Generation

**Overview:**  
Generates a unique certificate ID and a corresponding verification URL, then creates a QR code image URL linking to that verification page.

**Nodes Involved:**  
- Generate Certificate ID & QR (Function node)

**Node Details:**

- **Generate Certificate ID & QR**  
  - Type: Function  
  - Role: Creates a unique certificate identifier and a verification URL; generates QR code URL embedding this verification URL.  
  - Configuration:  
    - Certificate ID format: `CERT-CEU-{timestamp}-{randomString}`  
    - Verification URL: `https://training.verify.com/cert?id={certId}`  
    - QR Code URL generated using `qrserver.com` API with 300x300 PNG output.  
  - Outputs: JSON including original body fields plus `certificateId`, `verificationUrl`, `qrCodeUrl`, and generation timestamp; binary data includes QR code image auto-downloaded by n8n.  
  - Edge Cases: Randomness collisions are extremely unlikely; network errors fetching QR code image.

---

#### 1.3 Certificate Rendering & PDF Generation

**Overview:**  
Uses HTML and CSS to render a visually styled CEU certificate including recipient info, course details, CEUs, QR code, and issuance data. Converts this HTML into a PDF document.

**Nodes Involved:**  
- HTML to PDF  
- Download File

**Node Details:**

- **HTML to PDF**  
  - Type: HTML to PDF node (via external API)  
  - Role: Converts a rich HTML certificate template into a PDF file and returns a downloadable URL.  
  - Configuration:  
    - HTML includes placeholders for certificate data and QR code image embedded dynamically from previous node outputs.  
    - Custom fonts and styles for polished certificate appearance.  
  - Credentials: Requires HTMLCSSToPDF API credentials.  
  - Outputs: JSON containing `pdf_url` link to the generated PDF.  
  - Edge Cases: API failures, malformed HTML, rate limits.

- **Download File**  
  - Type: HTTP Request  
  - Role: Downloads the PDF file from the URL provided by the previous node.  
  - Configuration: Sets response format to file (binary).  
  - Outputs: Binary data of the PDF file for subsequent upload.  
  - Edge Cases: Network errors, file not found.

---

#### 1.4 File Handling & Storage

**Overview:**  
Uploads the generated PDF certificate file to a specified Google Drive folder, assigning a descriptive filename.

**Nodes Involved:**  
- Upload to Google Drive

**Node Details:**

- **Upload to Google Drive**  
  - Type: Google Drive  
  - Role: Uploads the binary PDF file with a filename pattern including recipient name and certificate ID.  
  - Configuration:  
    - Filename template: `CEU_Certificate_{name}_{certificateId}.png` (note: extension .png likely a minor oversight, actual file is PDF)  
    - Drive: "My Drive"  
    - Folder: Configured folder (default 'root' or specified folder)  
  - Credentials: Requires Google Drive OAuth2 credentials.  
  - Outputs: JSON with file metadata including `webViewLink` URL for sharing.  
  - Edge Cases: Google API quota limits, permission errors.

---

#### 1.5 Notification & Logging

**Overview:**  
Sends the certificate PDF via email to the recipient, notifies organizers on Slack with certificate details and PDF link, and logs all relevant data to a Google Sheets spreadsheet for audit trail.

**Nodes Involved:**  
- Send Email (Gmail)  
- Notify Organizers (Slack)  
- Log to Google Sheets

**Node Details:**

- **Send Email**  
  - Type: Gmail  
  - Role: Sends a personalized HTML email with certificate details and download link to the recipient’s email.  
  - Configuration:  
    - To: recipient email from webhook data.  
    - Subject: includes course title.  
    - Body: styled HTML with certificate summary, download button linking to PDF, and issuance info.  
  - Credentials: Requires Gmail OAuth2 credentials.  
  - Edge Cases: Gmail API rate limits, invalid email addresses, spam filtering.

- **Notify Organizers (Slack)**  
  - Type: Slack  
  - Role: Posts a notification message to a configured Slack channel announcing new certificate issuance with key data and PDF link.  
  - Configuration: Channel ID and message content templated with certificate info.  
  - Credentials: Slack OAuth2 credentials required.  
  - Edge Cases: Slack API permissions, invalid channel ID.

- **Log to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends a new row to the audit log sheet with all certificate and recipient data for tracking.  
  - Configuration:  
    - Columns mapped explicitly: Name, Email, Status, PDF URL, Timestamp, CEUs Earned, Course Title, Certificate ID, Training Hours, Completion Date, Instructor Name, Verification URL, Company/Provider.  
    - Document and sheet ID configured via credentials and parameters.  
  - Credentials: Google Sheets OAuth2 credentials.  
  - Edge Cases: Sheet access permission errors, document not found, API quota.

---

#### 1.6 Final Response

**Overview:**  
Responds back to the original webhook call with a success JSON payload including certificate ID, PDF URL, and Google Drive file URL.

**Nodes Involved:**  
- Respond to Webhook

**Node Details:**

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Returns HTTP 200 success response with JSON containing certificate metadata.  
  - Configuration: JSON includes success status, certificate ID, PDF URL from HTML to PDF node, and Google Drive link from upload node.  
  - Inputs: Triggered after logging to Google Sheets.  
  - Edge Cases: None significant here unless prior nodes fail to produce expected data.

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                          | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                              |
|----------------------------|-----------------------------|----------------------------------------|----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook Trigger            | Webhook Trigger             | Entry point to receive POST data       | -                          | Validate Required Fields     | # Validation  Accepts POST from any form/tool; checks required fields; validates email deliverability.  |
| Validate Required Fields   | If                         | Checks all required fields present     | Webhook Trigger            | Verifi Email; Stop: Incomplete Data | # Validation  Checks all required fields are present and non-empty.                                    |
| Stop: Incomplete Data      | Respond to Webhook          | Stops workflow if required data missing| Validate Required Fields   | -                           | # Validation  Returns HTTP 400 error on incomplete data.                                               |
| Verifi Email              | VerifiEmail                | Validates email deliverability         | Validate Required Fields   | Check Domain Verified        | # Validation  Uses VerifiEmail API to verify email address.                                            |
| Check Domain Verified      | If                         | Branches on email validity             | Verifi Email               | Generate Certificate ID & QR; Stop: Domain Not Verified | # Validation  Passes only if email is verified valid.                                                  |
| Stop: Domain Not Verified  | Respond to Webhook          | Stops workflow if email invalid        | Check Domain Verified      | -                           | # Validation  Returns HTTP 400 for invalid/disposable email.                                          |
| Generate Certificate ID & QR | Function                   | Generates unique ID and QR code URL    | Check Domain Verified (true)| HTML to PDF                  | # Generation  Generates unique certificate ID, verification URL, and QR code.                          |
| HTML to PDF                | HTML to PDF                 | Converts HTML certificate to PDF       | Generate Certificate ID & QR| Download File                | # Generation  Renders professional PDF certificate from HTML template.                                 |
| Download File              | HTTP Request                | Downloads generated PDF file            | HTML to PDF                | Upload to Google Drive       | # Generation  Downloads PDF via URL for upload.                                                        |
| Upload to Google Drive     | Google Drive                | Uploads PDF to Google Drive             | Download File              | Send Email                  | # Delivery  Upload certificate PDF to Google Drive.                                                    |
| Send Email                 | Gmail                       | Sends certificate email to recipient   | Upload to Google Drive     | Notify Organizers (Slack)    | # Delivery  Sends personalized certificate email with PDF link.                                       |
| Notify Organizers (Slack)  | Slack                       | Sends Slack notification to organizers | Send Email                 | Log to Google Sheets         | # Delivery  Sends Slack notification of new certificate.                                              |
| Log to Google Sheets       | Google Sheets               | Logs certificate issuance data         | Notify Organizers (Slack)  | Respond to Webhook           | # Delivery  Appends certificate data to audit log spreadsheet.                                        |
| Respond to Webhook         | Respond to Webhook          | Final success response to webhook call | Log to Google Sheets       | -                           | # Delivery  Returns success JSON with certificate ID and URLs.                                        |
| Sticky Note                | Sticky Note                 | Documentation note                     | -                          | -                           | # How It Works  Overview of workflow features and purpose.                                            |
| Sticky Note1               | Sticky Note                 | Documentation note                     | -                          | -                           | # Setup Credentials  One-time setup instructions for API keys and configuration.                       |
| Sticky Note2               | Sticky Note                 | Documentation note                     | -                          | -                           | # Validation  Explains data reception and email validation steps.                                     |
| Sticky Note3               | Sticky Note                 | Documentation note                     | -                          | -                           | # Generation  Describes certificate ID generation and PDF rendering.                                  |
| Sticky Note4               | Sticky Note                 | Documentation note                     | -                          | -                           | # Delivery  Details on uploading, emailing, Slack notification, and logging.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger node:**  
   - Type: Webhook Trigger  
   - HTTP Method: POST  
   - Path: `ceu-certificate`  
   - Response Mode: Use response node  
   - Purpose: Accept certificate request JSON payloads.

2. **Add If node "Validate Required Fields":**  
   - Check that `body.name`, `body.email`, `body.courseTitle`, `body.CEUsEarned`, and `body.completionDate` are all non-empty strings.  
   - Connect output TRUE to next step; FALSE to error response node.

3. **Add Respond to Webhook node "Stop: Incomplete Data":**  
   - Response Code: 400  
   - Respond With: JSON  
   - Response Body: `{"success": false, "error": "Missing required fields", "message": "Please provide name, email, courseTitle, CEUsEarned, and completionDate"}`  
   - Connect from FALSE output of validation node.

4. **Add Verifi Email node:**  
   - Input email: `{{$json.body.email}}`  
   - Credentials: Setup VerifiEmail API credentials from https://verifi.email  
   - Connect from TRUE output of validation node.

5. **Add If node "Check Domain Verified":**  
   - Condition: `$json.valid` equals `true`  
   - Connect TRUE to certificate generation; FALSE to domain error response.

6. **Add Respond to Webhook node "Stop: Domain Not Verified":**  
   - Response Code: 400  
   - Respond With: JSON  
   - Response Body: `{"success": false, "error": "Invalid email", "message": "The provided email address could not be verified or is from a disposable domain"}`  
   - Connect from FALSE output of domain verification node.

7. **Add Function node "Generate Certificate ID & QR":**  
   - Paste function code that:  
     - Generates unique certificate ID: `"CERT-CEU-" + current timestamp + random 6-char uppercase string`  
     - Creates verification URL: `https://training.verify.com/cert?id={certId}`  
     - Generates QR code URL from `qrserver.com` with verification URL embedded  
     - Returns updated JSON plus binary data for QR code image (auto-download).  
   - Connect from TRUE output of domain verification node.

8. **Add HTML to PDF node:**  
   - Paste the provided HTML template with placeholders for certificate data and QR code URL.  
   - Use credentials for HTMLCSSToPDF API.  
   - Connect from function node.

9. **Add HTTP Request node "Download File":**  
   - URL: `{{$json.pdf_url}}` (from previous node)  
   - Response Format: File (binary)  
   - Connect from HTML to PDF node.

10. **Add Google Drive node "Upload to Google Drive":**  
    - Upload Binary Data: PDF file from previous node  
    - File Name: `CEU_Certificate_{{name with spaces replaced by underscores}}_{{certificateId}}.png` (adjust extension to `.pdf`)  
    - Drive: Select appropriate drive (e.g., My Drive)  
    - Folder ID: Specify target folder (default root or custom)  
    - Credentials: Google Drive OAuth2  
    - Connect from Download File node.

11. **Add Gmail node "Send Email":**  
    - To: `{{$json.body.email}}`  
    - Subject: `✅ Your CEU Certificate - {{ $json.body.courseTitle }}`  
    - Message: Paste provided HTML email body template, replacing placeholders accordingly.  
    - Credentials: Gmail OAuth2  
    - Connect from Google Drive upload node.

12. **Add Slack node "Notify Organizers (Slack)":**  
    - Channel: Set your Slack channel ID  
    - Message: Use templated message with certificate details and PDF URL  
    - Credentials: Slack OAuth2  
    - Connect from Send Email node.

13. **Add Google Sheets node "Log to Google Sheets":**  
    - Operation: Append  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: Your target sheet/tab name  
    - Columns: Map all relevant certificate and recipient fields as described in the workflow.  
    - Credentials: Google Sheets OAuth2  
    - Connect from Slack notification node.

14. **Add Respond to Webhook node "Respond to Webhook":**  
    - Response JSON:  
      ```json
      {
        "success": true,
        "message": "CEU Certificate generated and sent successfully",
        "certificateId": "{{ $json.certificateId }}",
        "certificateUrl": "{{ $json.pdf_url }}",
        "driveUrl": "{{ $json.webViewLink }}"
      }
      ```  
    - Connect from Google Sheets node.

15. **Optional Sticky Notes:** Add notes to document workflow blocks and setup instructions as per your preference.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Verified Corporate Training Certificate with CEUs: Automates secure, verifiable CEU certificate generation.    | General workflow purpose and feature summary.                                                   |
| Setup Credentials: Verifi.Email API (https://verifi.email), HTML to PDF API (https://pdfmunk.com), Google APIs | Required credentials and setup for API integrations.                                            |
| Update webhook URL in your forms and customize company name/logo in the HTML template for branding.             | Configuration instruction for deployment.                                                       |
| Slack notification and Google Sheets logging provide audit trail and team awareness.                            | Operational monitoring and record keeping.                                                      |
| QR Code verification uses https://api.qrserver.com for generating QR images linking to certificate verification.| External QR code generation service.                                                             |

---

This detailed reference document should enable advanced users or automation agents to fully understand, reproduce, customize, and troubleshoot the "Verified Corporate Training Certificate with CEUs" workflow.