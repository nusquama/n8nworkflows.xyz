Automatically Save Kindle Handwritten Notes to Google Drive with DeepSeek AI

https://n8nworkflows.xyz/workflows/automatically-save-kindle-handwritten-notes-to-google-drive-with-deepseek-ai-9860


# Automatically Save Kindle Handwritten Notes to Google Drive with DeepSeek AI

---

### 1. Workflow Overview

This workflow automates the retrieval and archival of handwritten Kindle notes exported as PDF files. Kindle devices like the Kindle Scribe allow handwritten notes to be exported, but the export process sends a unique, temporary URL via email rather than a direct attachment, complicating access and storage.

The workflow addresses this by:

- Monitoring a Gmail inbox for Kindle export notification emails.
- Parsing the email content to extract the dynamic PDF download URL using DeepSeek AI.
- Downloading the PDF file from the extracted URL.
- Uploading the PDF to a designated Google Drive folder.
- Sending a success notification email upon completion.

Logical blocks:

- **1.1 Input Reception:** Monitoring Gmail for specific Amazon Kindle export emails.
- **1.2 Email Content Processing:** Converting email body to HTML and extracting the PDF download URL.
- **1.3 AI-Driven URL Extraction:** Using DeepSeek AI to identify and return the precise download URL.
- **1.4 PDF Download:** Requesting and retrieving the PDF using the extracted URL.
- **1.5 Google Drive Upload and Notification:** Uploading the PDF to Google Drive and sending a confirmation email.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming emails from Amazon containing Kindle note exports. It filters emails by sender and subject to capture only relevant notifications.

**Nodes Involved:**  
- Email ingestion: Gmail trigger

**Node Details:**  

- **Email ingestion: Gmail trigger**  
  - Type: Gmail Trigger (listens for new emails)  
  - Configuration:  
    - Filter: Emails from amazon.com with subject containing "You sent a file"  
    - Includes Spam and Trash folders  
    - Poll interval: every minute  
  - Inputs: None (trigger node)  
  - Outputs: Email data object including headers and body  
  - Edge cases: Gmail API quota limits, authentication issues, email format changes  
  - Credentials: OAuth2 Gmail account  
  - Notes: The node must be authorized with a Gmail account having access to the target mailbox.

---

#### 1.2 Email Content Processing

**Overview:**  
This block converts the incoming email content to an HTML format and parses it into a table structure, preparing it for AI-based URL extraction.

**Nodes Involved:**  
- Link extraction: HTML parser

**Node Details:**  

- **Link extraction: HTML parser**  
  - Type: HTML Parser  
  - Configuration:  
    - Operation: Convert email content to an HTML table representation  
  - Inputs: Output from Gmail Trigger (raw email content)  
  - Outputs: Parsed HTML table data containing the email's structure and content  
  - Edge cases: Malformed or unexpected email HTML structure, empty or missing email body  
  - Version: 1.2  
  - Notes: This node simplifies complex email content for easier AI processing.

---

#### 1.3 AI-Driven URL Extraction

**Overview:**  
This block uses DeepSeek’s AI-powered language model to analyze the parsed email content and extract the unique, dynamic URL for the PDF download.

**Nodes Involved:**  
- DeepSeek Chat Model  
- AI Agent - Link Extraction

**Node Details:**  

- **DeepSeek Chat Model**  
  - Type: DeepSeek Language Model Chat Node  
  - Configuration:  
    - Model: deepseek-reasoner  
    - Options: Default  
  - Inputs: None directly; credentials assigned  
  - Outputs: AI-generated response with URL extraction logic  
  - Credentials: DeepSeek API account  
  - Edge cases: API rate limits, connection failures, inaccurate URL extraction if email format changes

- **AI Agent - Link Extraction**  
  - Type: Langchain AI Agent (specialized for link extraction)  
  - Configuration:  
    - Task description specifies extracting a URL hidden inside a specific HTML tag pattern  
    - Input: Parsed HTML table data from previous node  
    - Output: Plain URL string only, no formatting or extra text  
  - Inputs: Parsed HTML content from the HTML parser node  
  - Outputs: Extracted URL string  
  - Edge cases: AI misinterpretation, unexpected HTML changes, incomplete data passed  
  - Version: 1.9  
  - Notes: Uses a defined prompt to reliably extract the URL, avoiding fragile regex patterns.

---

#### 1.4 PDF Download

**Overview:**  
This block downloads the PDF file using the URL extracted by the AI agent.

**Nodes Involved:**  
- PDF retrieval - HTTP request

**Node Details:**  

- **PDF retrieval - HTTP request**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: Dynamically set from AI Agent output (the extracted URL)  
    - Method: GET (default)  
  - Inputs: Extracted URL string  
  - Outputs: Binary PDF data  
  - Edge cases: URL expiration, HTTP request failures, network timeouts, invalid URLs  
  - Version: 4.2  
  - Notes: The request must handle temporary URLs and possible security tokens embedded in the URL.

---

#### 1.5 Google Drive Upload and Notification

**Overview:**  
This block uploads the downloaded PDF to a predefined Google Drive folder and sends an email notification upon success.

**Nodes Involved:**  
- File upload - Google Drive  
- Success notification - Email

**Node Details:**  

- **File upload - Google Drive**  
  - Type: Google Drive node  
  - Configuration:  
    - File name: Set dynamically from the original email subject line  
    - Drive: "My Drive"  
    - Folder ID: Configured folder for Kindle Notes (specific folder identified by ID)  
  - Inputs: Binary PDF data from HTTP request node  
  - Outputs: Confirmation of file upload  
  - Credentials: OAuth2 Google Drive account  
  - Edge cases: Upload failures, permission issues, quota limits

- **Success notification - Email**  
  - Type: Gmail Send Email node  
  - Configuration:  
    - Recipient: Predefined email address (user’s Gmail)  
    - Subject: "KINDLE NOTE SUCCESSFULLY UPLOADED"  
    - Message: Includes uploaded file name dynamically from JSON data  
    - Behavior: Executes only if all previous steps succeed  
  - Inputs: Confirmation output from Google Drive upload  
  - Outputs: Email sent confirmation  
  - Credentials: OAuth2 Gmail account  
  - Edge cases: Email sending failures, misconfiguration of recipient address  
  - Additional: Set to continue execution on error to prevent workflow failure if notification fails.

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                           | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                       |
|------------------------------|-----------------------------------|-----------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------|
| Email ingestion: Gmail trigger| Gmail Trigger                     | Capture Kindle export notification emails| None                        | Link extraction: HTML parser | ## Step #1: Emails 1. Configure Kindle to send exports to gmail. 2. Configure gmail node to react to specific mails. |
| Link extraction: HTML parser  | HTML Parser                      | Convert email content to HTML table      | Email ingestion: Gmail trigger| AI Agent - Link Extraction   | ## Step #2: Convert to HTML 1. Convert email for URL extraction ease.                                           |
| AI Agent - Link Extraction    | Langchain AI Agent               | Extract dynamic PDF URL from HTML        | Link extraction: HTML parser | PDF retrieval - HTTP request | ## Step #3: Extract the URL using DeepSeek 1. Use AI to find the dynamic URL rather than regex.                   |
| DeepSeek Chat Model           | DeepSeek Language Model          | Supports AI Agent with reasoning         | None (credential node)       | AI Agent - Link Extraction   |                                                                                                                  |
| PDF retrieval - HTTP request  | HTTP Request                    | Download PDF from URL                     | AI Agent - Link Extraction   | File upload - Google Drive   | ## Step #4: Make a request to download those notes 1. Use extracted URL to grab PDF.                            |
| File upload - Google Drive    | Google Drive                    | Upload PDF to Kindle Notes folder         | PDF retrieval - HTTP request | Success notification - Email | ## Step #5: Upload the .pdf to GDrive 1. Upload .pdf for notes backup 2. Send success email notification.        |
| Success notification - Email  | Gmail Send Email                | Notify user of successful upload          | File upload - Google Drive   | None                        |                                                                                                                  |
| Sticky Note                  | Sticky Note                      | Documentation and instructions            | None                        | None                        | See notes in Sticky Note content below.                                                                          |
| Sticky Note1                 | Sticky Note                      | Background and challenge explanation      | None                        | None                        | See notes in Sticky Note1 content below.                                                                         |
| Sticky Note2                 | Sticky Note                      | Step 2 instructions                        | None                        | None                        | See notes in Sticky Note2 content below.                                                                         |
| Sticky Note3                 | Sticky Note                      | Step 3 instructions                        | None                        | None                        | See notes in Sticky Note3 content below.                                                                         |
| Sticky Note4                 | Sticky Note                      | Step 4 instructions                        | None                        | None                        | See notes in Sticky Note4 content below.                                                                         |
| Sticky Note5                 | Sticky Note                      | Step 5 instructions                        | None                        | None                        | See notes in Sticky Note5 content below.                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Node Type: Gmail Trigger  
   - Configure OAuth2 credentials for your Gmail account.  
   - Set filter: `from:(amazon.com) subject:(You sent a file)`  
   - Include spam and trash folders.  
   - Poll every minute.

2. **Add HTML Parser Node**  
   - Node Type: HTML Parser  
   - Connect input from Gmail Trigger node output.  
   - Set operation: Convert email to HTML table.

3. **Configure DeepSeek Chat Model Node**  
   - Node Type: DeepSeek Language Model Chat  
   - Use DeepSeek API credentials.  
   - Select model “deepseek-reasoner”.  
   - No additional options needed.

4. **Add AI Agent - Link Extraction Node**  
   - Node Type: Langchain AI Agent  
   - Connect input from HTML Parser node output.  
   - Configure prompt to extract URL from HTML content with instructions:  
     - Act as frontend engineer.  
     - Extract URL hidden inside `<span class="rio-text rio-text-31"><a href="`.  
     - Return only the URL string, no formatting.

5. **Connect DeepSeek Chat Model node output to AI Agent - Link Extraction input**  
   - This supports the AI extraction process.

6. **Add HTTP Request Node for PDF Download**  
   - Node Type: HTTP Request  
   - Connect input from AI Agent output.  
   - Set URL parameter to the extracted URL from AI Agent (`={{ $json.output }}`).  
   - Use GET request.  
   - No special headers required unless URL mandates.

7. **Add Google Drive Upload Node**  
   - Node Type: Google Drive  
   - Connect input from HTTP Request output (binary PDF).  
   - Configure OAuth2 credentials for Google Drive.  
   - Set file name dynamically from email subject `={{ $('Email ingestion: Gmail trigger').item.json.headers.subject }}`.  
   - Choose “My Drive” as drive.  
   - Set target folder by folder ID (e.g., Kindle Notes folder ID).

8. **Add Gmail Send Email Node for Success Notification**  
   - Node Type: Gmail  
   - Connect input from Google Drive node output.  
   - Configure OAuth2 Gmail credentials.  
   - Set recipient email address.  
   - Subject: "KINDLE NOTE SUCCESSFULLY UPLOADED"  
   - Message includes file name: `={{ $json.name }}`  
   - Set onError behavior to “continueRegularOutput” to avoid stopping workflow.

9. **Sequence connections:**  
   - Gmail Trigger → HTML Parser → AI Agent → HTTP Request → Google Drive Upload → Success Email  
   - DeepSeek Chat Model node linked as AI Language Model provider to AI Agent node.

10. **Add Sticky Notes** for clarity and future maintenance with the detailed instructions and explanations as in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| The workflow addresses the challenge that Kindle handwritten note exports are not stored centrally but sent as unique temporary URLs via email. Automating retrieval improves user experience.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Kindle Note Export Automation Background (Sticky Note1)                                                                     |
| To configure your Kindle device for exporting notes, set the device to send exports to your Gmail address. Then configure the Gmail trigger to listen for these specific emails.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Step #1 Instructions (Sticky Note)                                                                                          |
| Converting the email content to HTML simplifies extraction of the download URL, which is embedded in a complex HTML email structure.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Step #2 Instructions (Sticky Note2)                                                                                         |
| DeepSeek AI is leveraged instead of regex to reliably extract the dynamic URL from the email’s complex HTML content.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Step #3 Instructions (Sticky Note3)                                                                                         |
| The HTTP request node downloads the PDF from the extracted URL, which is time-sensitive. The node must handle potential download failures if the URL expires or is invalid.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Step #4 Instructions (Sticky Note4)                                                                                         |
| The PDF is uploaded to a dedicated Google Drive folder for Kindle notes, ensuring safe, centralized storage. Upon success, an email notification is sent to the user.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Step #5 Instructions (Sticky Note5)                                                                                         |
| DeepSeek AI credentials and Google OAuth2 credentials must be properly configured prior to workflow use. Gmail OAuth2 credentials require appropriate scope for triggers and sending emails.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Credential requirements                                                                                                    |
| Refer to DeepSeek AI documentation for advanced prompt engineering to customize URL extraction if email formats change.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | https://docs.deepseek.ai                                                                                                    |
| Google Drive folder ID must be set to a folder where you have upload permissions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Google Drive folder setup                                                                                                  |

---

**Disclaimer:** This text exclusively derives from an automated workflow created using n8n, an integration and automation platform. All data processed is legal and public. The workflow complies fully with applicable content policies and contains no illegal, offensive, or protected elements.

---