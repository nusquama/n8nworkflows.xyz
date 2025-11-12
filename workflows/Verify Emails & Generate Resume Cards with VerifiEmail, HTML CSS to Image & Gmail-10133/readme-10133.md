Verify Emails & Generate Resume Cards with VerifiEmail, HTML CSS to Image & Gmail

https://n8nworkflows.xyz/workflows/verify-emails---generate-resume-cards-with-verifiemail--html-css-to-image---gmail-10133


# Verify Emails & Generate Resume Cards with VerifiEmail, HTML CSS to Image & Gmail

### 1. Workflow Overview

This workflow automates the process of verifying candidate email addresses and generating personalized resume snapshot cards delivered via email. It is designed primarily for recruitment or HR use cases where validating contact information and providing visually appealing candidate summaries is critical.

The workflow consists of the following logical blocks:

- **1.1 Webhook Trigger:** Receives candidate resume data as an HTTP POST payload.
- **1.2 Data Extraction & Preparation:** Extracts and cleans key fields from the incoming JSON.
- **1.3 Email Verification:** Validates the provided email address using the VerifiEmail API, checking format, domain, mailbox existence, and disposability.
- **1.4 Conditional Routing:** Routes the workflow based on email validity; valid emails proceed to card generation, invalid emails trigger a failure notification.
- **1.5 Data Merge:** Combines the cleaned resume data and verification results into one object.
- **1.6 Visual Resume Card Generation:** Creates a styled image of the resume card using htmlcsstoimage.com.
- **1.7 Image Download:** Downloads the generated PNG image for attachment.
- **1.8 Email Delivery:** Sends the resume card to valid addresses or sends an invalid email notice.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Trigger

- **Overview:**  
  Receives incoming HTTP POST requests containing candidate resume information (name, email, role, skills). Acts as the entry point for the workflow.

- **Nodes Involved:**  
  - Webhook  
  - Sticky Note - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: HTTP Webhook  
    - Configuration: Accepts POST requests at path `/resume-verifier`, responds with the output of the last node executed.  
    - Input: External HTTP POST payload (JSON with keys: name, email, role, skills).  
    - Output: Passes entire body to next node.  
    - Edge cases: Missing or malformed payloads may cause errors or empty data downstream.  
  - **Sticky Note - Webhook**  
    - Purpose: Documentation of expected payload format and test instructions.  
    - No technical impact.

#### 1.2 Data Extraction & Preparation

- **Overview:**  
  Extracts relevant fields (`name`, `email`, `role`, `skills`) from the webhook payload and formats them as individual fields for downstream processing.

- **Nodes Involved:**  
  - Set - Prepare Resume Data  
  - Sticky Note - Set Node

- **Node Details:**  
  - **Set - Prepare Resume Data**  
    - Type: Set node  
    - Configuration: Maps `$json.body.name` → `name`, `$json.body.email` → `email`, `$json.body.role` → `role`, `$json.body.skills` → `skills`.  
    - Input: Raw webhook JSON.  
    - Output: Cleaned, structured JSON with four fields.  
    - Edge cases: Missing fields in payload result in empty values; no validation here.  
  - **Sticky Note - Set Node**  
    - Describes field mappings for clarity.

#### 1.3 Email Verification

- **Overview:**  
  Uses VerifiEmail API to validate the email address on multiple criteria: RFC format, MX record existence, mailbox deliverability, spoof and disposable detection.

- **Nodes Involved:**  
  - Verifi Email  
  - Sticky Note - Email Verification

- **Node Details:**  
  - **Verifi Email**  
    - Type: VerifiEmail API node  
    - Configuration: Email input set dynamically to `{{$json.email}}` from previous Set node.  
    - Credentials: Requires VerifiEmail API Key.  
    - Output: JSON with `valid` boolean and detailed verification metadata.  
    - Edge cases: API failures, rate limits, invalid API key, network timeouts.  
  - **Sticky Note - Email Verification**  
    - Explains the various verification checks performed.

#### 1.4 Conditional Routing

- **Overview:**  
  Routes the workflow based on whether the email is valid (`valid === true`). Valid emails follow the path to generate the resume card and send it. Invalid emails get a failure notification email.

- **Nodes Involved:**  
  - IF - Check Email Valid  
  - Sticky Note - IF Logic

- **Node Details:**  
  - **IF - Check Email Valid**  
    - Type: IF node  
    - Configuration: Condition checks if `$json.valid === true`.  
    - Output 1 (true branch): Continue with merge and card generation.  
    - Output 2 (false branch): Send invalid email notice.  
    - Edge cases: If `valid` field is missing or malformed, may default to false branch or cause errors.  
  - **Sticky Note - IF Logic**  
    - Documents the routing logic.

#### 1.5 Data Merge

- **Overview:**  
  Combines the cleaned resume data and the verification result into a single JSON object for use in card generation.

- **Nodes Involved:**  
  - Merge - Combine Data  
  - Sticky Note - Merge

- **Node Details:**  
  - **Merge - Combine Data**  
    - Type: Merge node  
    - Configuration: Combines inputs by position from "Set - Prepare Resume Data" and output from IF node's true branch.  
    - Output: Merged JSON containing resume fields plus verification details.  
    - Edge cases: Mismatched input counts could cause empty merges.  
  - **Sticky Note - Merge**  
    - Shows example inputs and merged output.

#### 1.6 Visual Resume Card Generation

- **Overview:**  
  Generates a styled PNG image of the resume card using HTML and CSS rendered by htmlcsstoimage.com.

- **Nodes Involved:**  
  - Generate Image  
  - Sticky Note - Image Generation

- **Node Details:**  
  - **Generate Image**  
    - Type: HTML/CSS to Image node  
    - Configuration:  
      - HTML template uses variables `{{$json.name}}`, `{{$json.role}}`, `{{$json.skills}}`, `{{$json.email}}`.  
      - Styles include gradients, shadows, and verification badge.  
    - Credentials: Requires htmlcsstoimage.com API User ID and API Key.  
    - Output: JSON with `image_url` pointing to generated PNG.  
    - Edge cases: API failures, malformed HTML, timeout, credential errors.  
  - **Sticky Note - Image Generation**  
    - Lists input data fields used.

#### 1.7 Image Download

- **Overview:**  
  Downloads the PNG image from the URL returned by the previous node, storing it as binary data for email attachment.

- **Nodes Involved:**  
  - Download Image  
  - Sticky Note - HTTP Request

- **Node Details:**  
  - **Download Image**  
    - Type: HTTP Request node  
    - Configuration:  
      - URL set dynamically to `{{$json.image_url}}`.  
      - Response format set to "file" to capture binary data.  
    - Output: Binary data object containing PNG image.  
    - Edge cases: Download failures, invalid/missing URL, network errors.  
  - **Sticky Note - HTTP Request**  
    - Explains the process of image download and storage.

#### 1.8 Email Delivery

- **Overview:**  
  Sends the personalized resume card image via Gmail to valid email addresses or sends a failure notification email to invalid addresses.

- **Nodes Involved:**  
  - Gmail - Send Valid Card  
  - Sticky Note - Gmail Valid  
  - Gmail - Send Invalid Notice  
  - Sticky Note - Gmail Invalid

- **Node Details:**  
  - **Gmail - Send Valid Card**  
    - Type: Gmail node (OAuth2)  
    - Configuration:  
      - Sends to email from `Set - Prepare Resume Data`.  
      - HTML email content personalized with candidate name and verification success message.  
      - Attaches binary PNG image from "Download Image" node.  
      - Subject: "Your Verified Resume Snapshot Card 🎉"  
    - Credentials: Gmail OAuth2 required.  
    - Edge cases: OAuth token expiry, attachment size limits, SMTP errors.  
  - **Sticky Note - Gmail Valid**  
    - Describes email features and attachment details.  
  - **Gmail - Send Invalid Notice**  
    - Type: Gmail node (OAuth2)  
    - Configuration:  
      - Sends to candidate email with failure message and troubleshooting tips.  
      - No attachments.  
      - Subject: "Email Verification Failed ⚠️"  
    - Credentials: Gmail OAuth2 required.  
    - Edge cases: Same as above.  
  - **Sticky Note - Gmail Invalid**  
    - Describes common failure reasons and error handling benefits.

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                  | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                          |
|---------------------------|-----------------------|--------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook                   | Webhook               | Entry point, receive payload   | -                                | Set - Prepare Resume Data         | ## 🟩 WEBHOOK TRIGGER Expected Payload example and test instructions                                 |
| Sticky Note - Webhook      | Sticky Note           | Documentation                  | -                                | -                                | ## 🟩 WEBHOOK TRIGGER Expected Payload example and test instructions                                 |
| Sticky Note - Credentials Setup | Sticky Note       | Documentation for credentials  | -                                | -                                | ## ⚙️ CREDENTIALS SETUP GUIDE Details for Gmail OAuth2, VerifiEmail API, HTML/CSS to Image API       |
| Set - Prepare Resume Data  | Set                   | Extract and format input data  | Webhook                         | Verifi Email, Merge - Combine Data| ## 🟨 DATA EXTRACTION & CLEANING Field mappings                                                      |
| Sticky Note - Set Node     | Sticky Note           | Documentation                  | -                                | -                                | ## 🟨 DATA EXTRACTION & CLEANING Field mappings                                                      |
| Verifi Email              | VerifiEmail API node   | Email validation               | Set - Prepare Resume Data        | IF - Check Email Valid            | ## 🟦 EMAIL VERIFICATION SERVICE Explains checks performed                                          |
| Sticky Note - Email Verification | Sticky Note       | Documentation                  | -                                | -                                | ## 🟦 EMAIL VERIFICATION SERVICE Explains checks performed                                          |
| IF - Check Email Valid     | IF                    | Conditional routing            | Verifi Email                    | Merge - Combine Data (true), Gmail - Send Invalid Notice (false) | ## 🟧 CONDITIONAL ROUTING LOGIC Condition & flow explanation                                         |
| Sticky Note - IF Logic     | Sticky Note           | Documentation                  | -                                | -                                | ## 🟧 CONDITIONAL ROUTING LOGIC Condition & flow explanation                                         |
| Merge - Combine Data       | Merge                  | Combine resume and validation  | Set - Prepare Resume Data, IF    | Generate Image                   | ## 🔀 DATA MERGE OPERATION Shows input and merged output examples                                   |
| Sticky Note - Merge        | Sticky Note           | Documentation                  | -                                | -                                | ## 🔀 DATA MERGE OPERATION Shows input and merged output examples                                   |
| Generate Image             | HTML/CSS to Image     | Render visual resume card      | Merge - Combine Data             | Download Image                   | ## 🟪 VISUAL CARD GENERATION Lists input data fields                                                |
| Sticky Note - Image Generation | Sticky Note        | Documentation                  | -                                | -                                | ## 🟪 VISUAL CARD GENERATION Lists input data fields                                                |
| Download Image             | HTTP Request          | Download generated PNG image   | Generate Image                   | Gmail - Send Valid Card           | ## 📥 IMAGE DOWNLOAD Explains image download process                                                |
| Sticky Note - HTTP Request | Sticky Note           | Documentation                  | -                                | -                                | ## 📥 IMAGE DOWNLOAD Explains image download process                                                |
| Gmail - Send Valid Card    | Gmail                 | Send verified resume card email| Download Image                  | -                                | ## 🟪 SUCCESS EMAIL DELIVERY Describes email content and attachment                                 |
| Sticky Note - Gmail Valid  | Sticky Note           | Documentation                  | -                                | -                                | ## 🟪 SUCCESS EMAIL DELIVERY Describes email content and attachment                                 |
| Gmail - Send Invalid Notice| Gmail                 | Send invalid email notification| IF - Check Email Valid (false)  | -                                | ## 🟥 INVALID EMAIL HANDLER Lists failure reasons and error handling benefits                       |
| Sticky Note - Gmail Invalid| Sticky Note           | Documentation                  | -                                | -                                | ## 🟥 INVALID EMAIL HANDLER Lists failure reasons and error handling benefits                       |
| Sticky Note - Workflow Overview | Sticky Note       | Overall workflow description  | -                                | -                                | ## 📊 WORKFLOW OVERVIEW Summarizes purpose, steps, and logic                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `resume-verifier`  
   - Response Mode: Last Node  
   - No authentication required  
   - Purpose: To receive incoming resume data JSON with fields: name, email, role, skills.

2. **Add Set Node: Prepare Resume Data**  
   - Type: Set  
   - Add fields:  
     - `name` → Expression: `{{$json["body"]["name"]}}`  
     - `email` → Expression: `{{$json["body"]["email"]}}`  
     - `role` → Expression: `{{$json["body"]["role"]}}`  
     - `skills` → Expression: `{{$json["body"]["skills"]}}`  
   - Connect Webhook output to this node.

3. **Add VerifiEmail Node**  
   - Type: VerifiEmail API node  
   - Configure credentials: Use your VerifiEmail API Key (create credential in n8n first).  
   - Email field: Expression: `{{$json["email"]}}`  
   - Connect Set node output to this node.

4. **Add IF Node: Check Email Valid**  
   - Condition: Boolean check where `{{$json["valid"]}}` equals `true`  
   - Connect VerifiEmail node output to IF node.

5. **Add Merge Node: Combine Data**  
   - Mode: Combine  
   - Combination Mode: Merge by Position (default)  
   - Connect Set node output to first input of Merge node.  
   - Connect IF node's TRUE output to second input of Merge node.

6. **Add HTML/CSS to Image Node: Generate Image**  
   - Configure API credentials: Use your htmlcsstoimage.com User ID and API Key.  
   - HTML content: Use a template that references variables:  
     - `{{$json.name}}`, `{{$json.role}}`, `{{$json.skills}}`, `{{$json.email}}`  
   - Connect Merge node output to this node.

7. **Add HTTP Request Node: Download Image**  
   - Method: GET  
   - URL: Expression: `{{$json["image_url"]}}`  
   - Response Format: File (to get binary data)  
   - Connect Generate Image node output to this node.

8. **Add Gmail Node: Send Valid Card**  
   - Configure Gmail OAuth2 credentials (create credential in n8n).  
   - To: Expression: `{{$json["email"]}}` from Set node data.  
   - Subject: "Your Verified Resume Snapshot Card 🎉"  
   - Message: Use personalized HTML with the candidate's name and a message about verification success.  
   - Attachments: Attach binary data from Download Image node (field: data).  
   - Connect Download Image node output to this node.

9. **Add Gmail Node: Send Invalid Notice**  
   - Configure Gmail OAuth2 credentials.  
   - To: Expression: `{{$json["email"]}}` from Set node data.  
   - Subject: "Email Verification Failed ⚠️"  
   - Message: HTML content explaining failure and troubleshooting tips.  
   - Connect IF node FALSE output to this node.

10. **Connect Flow**  
    - Webhook → Set - Prepare Resume Data  
    - Set → VerifiEmail  
    - VerifiEmail → IF node  
    - IF TRUE → Merge (along with Set node) → Generate Image → Download Image → Gmail Send Valid Card  
    - IF FALSE → Gmail Send Invalid Notice

11. **Credential Setup**  
    - Gmail OAuth2: Create credentials with Google Cloud Console, enable Gmail API, obtain Client ID/Secret, configure scopes.  
    - VerifiEmail API Key: Sign up at https://verifi.email, retrieve API key.  
    - HTML/CSS to Image API: Sign up at https://htmlcsstoimg.com, get User ID and API Key.

12. **Testing**  
    - Use Postman or cURL to POST JSON payload to webhook URL:  
      ```json
      {
        "name": "John Doe",
        "email": "john@gmail.com",
        "role": "Frontend Developer",
        "skills": "React, JavaScript, Tailwind, Git"
      }
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Credential setup is mandatory for Gmail OAuth2, VerifiEmail API, and HTML/CSS to Image API to function.     | See Sticky Note - Credentials Setup              |
| Workflow uses professional HTML email templates with inline CSS for responsive design and branding.         | Gmail Send nodes email content                    |
| VerifiEmail API covers multiple verification layers: format, domain MX, mailbox deliverability, spoof check.| https://verifi.email                             |
| HTML/CSS to Image API generates PNG images from HTML markup with styling for visually appealing cards.      | https://htmlcsstoimg.com                         |
| Test webhook payloads with tools like Postman or cURL to simulate real-world usage.                         | Sticky Note - Webhook                             |
| Avoid silent failures by sending clear invalid email notifications with troubleshooting steps.              | Sticky Note - Gmail Invalid                        |
| The workflow is designed for scalability but watch API rate limits and credential expirations.              | General best practice                             |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.