Employee Attrition Risk Detection & HR Alerts using Azure OpenAI GPT-4o-mini & Gmail

https://n8nworkflows.xyz/workflows/employee-attrition-risk-detection---hr-alerts-using-azure-openai-gpt-4o-mini---gmail-9309


# Employee Attrition Risk Detection & HR Alerts using Azure OpenAI GPT-4o-mini & Gmail

### 1. Workflow Overview

This workflow automates the detection of employee attrition risk by analyzing newly received resumes in a monitored Google Drive folder. It uses Azure OpenAI GPT-4o-mini to extract and compute the average employment tenure from resumes, applies business logic to identify high attrition risk candidates, drafts alert emails to HR or managers, and sends them via Gmail. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Detect new resumes uploaded to a specific Google Drive folder and download the file.
- **1.2 Text Extraction:** Extract readable text from the downloaded PDF resume.
- **1.3 AI Processing:** Use Azure OpenAI GPT-4o-mini to analyze extracted text and calculate the average employment span in months.
- **1.4 Decision Logic:** Apply conditional checks on the average tenure to classify attrition risk.
- **1.5 Email Generation:** Construct personalized emails alerting HR or managers about high attrition risk employees.
- **1.6 Email Dispatch:** Send the composed emails via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initiates the workflow upon detection of a new resume file in a designated Google Drive folder and downloads the file for processing.
- **Nodes Involved:**  
  - Trigger for new resume  
  - Download resume

- **Node Details:**

  - **Trigger for new resume**  
    - Type: Google Drive Trigger  
    - Role: Watches a specific Google Drive folder for newly created files, triggering the workflow every minute.  
    - Configuration:  
      - Event: fileCreated  
      - Folder: Specified by folder ID `1KyX5RGqeF7v0sAvvaoBnJPTQoq4aOHLz`  
      - Poll interval: every minute  
    - Input: None (trigger)  
    - Output: Metadata including file webViewLink  
    - Potential Failures: Authentication errors with Google Drive OAuth2, connectivity issues, or folder access permissions.

  - **Download resume**  
    - Type: Google Drive (File download)  
    - Role: Downloads the resume file using the URL from the trigger node.  
    - Configuration:  
      - File ID sourced from the trigger node’s `webViewLink` URL field.  
      - Operation: download file content  
    - Input: File metadata from trigger  
    - Output: Binary file data for the resume  
    - Potential Failures: Invalid file ID, permission errors, download timeouts.

#### 2.2 Text Extraction

- **Overview:** Extracts the textual content from the downloaded PDF resume to enable AI processing.
- **Nodes Involved:**  
  - Extract text

- **Node Details:**

  - **Extract text**  
    - Type: Extract From File  
    - Role: Converts binary PDF resume into plain text.  
    - Configuration:  
      - Operation: pdf extraction  
    - Input: Binary file content from "Download resume"  
    - Output: JSON with extracted text under the `text` property  
    - Potential Failures: Corrupted PDF files, unsupported formats, extraction library limitations.

#### 2.3 AI Processing

- **Overview:** Uses Azure OpenAI GPT-4o-mini via Langchain integration to analyze the extracted resume text, parse employment experiences, and compute the average tenure in months.
- **Nodes Involved:**  
  - Azure OpenAI Chat Model  
  - Structured Output Parser  
  - Calculate avg span

- **Node Details:**

  - **Azure OpenAI Chat Model**  
    - Type: Langchain Azure OpenAI Chat Node  
    - Role: Sends extracted text to GPT-4o-mini for structured analysis.  
    - Configuration:  
      - Model: gpt-4o-mini  
      - No additional options specified  
    - Credentials: Azure Open AI API  
    - Input: Extracted text  
    - Output: Raw GPT response  
    - Potential Failures: API authentication issues, rate limits, timeouts, malformed prompts.

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses GPT response into structured JSON based on a schema example (e.g., `{ "average": "18" }`).  
    - Configuration:  
      - JSON schema example to extract average tenure  
    - Input: Raw GPT response from Chat Model  
    - Output: Parsed JSON with average tenure value  
    - Potential Failures: Parsing failures if GPT output format varies, schema mismatches.

  - **Calculate avg span**  
    - Type: Langchain Agent  
    - Role: Defines a system prompt instructing GPT to extract and compute average employment tenure in months from resume text with detailed parsing rules.  
    - Configuration:  
      - System message includes detailed instructions on date parsing, experience grouping, duration calculation, and averaging.  
      - Prompt type: define  
      - Output parser enabled to format final output.  
    - Input: Extracted text from previous node, GPT output, and parsed JSON  
    - Output: JSON object with `average` tenure in months  
    - Potential Failures: Ambiguous text in resumes, inconsistent date formats, incomplete date ranges.

#### 2.4 Decision Logic

- **Overview:** Applies a conditional check to determine if the average employment tenure is below 12 months, indicating potential high attrition risk.
- **Nodes Involved:**  
  - Logic (If node)

- **Node Details:**

  - **Logic**  
    - Type: If node  
    - Role: Compares average tenure numeric value to threshold of 12 months.  
    - Configuration:  
      - Condition: average tenure < 12 (strict number comparison)  
    - Input: JSON with average tenure  
    - Output: True branch if condition met (high attrition risk), False otherwise  
    - Potential Failures: Missing or non-numeric average field, expression evaluation errors.

#### 2.5 Email Generation

- **Overview:** Constructs a personalized email to alert HR or managers about the employee’s attrition risk, including risk level classification, signals, and recommended actions.
- **Nodes Involved:**  
  - Create email

- **Node Details:**

  - **Create email**  
    - Type: Code node (JavaScript)  
    - Role: Generates email subject, body, recipients, and metadata based on input JSON attributes such as employee name, role, risk score, signals, last engagement date, and recommended actions.  
    - Configuration:  
      - Implements helper functions for date formatting, risk score clamping (0-100), risk classification (High/Medium/Low), and bullet list formatting.  
      - Sets default recommended actions if none provided.  
      - Composes plain text email body and subject line with structured metadata for logging.  
    - Input: JSON item with risk score and employee details from Logic node  
    - Output: JSON with emailSubject, emailBody, emailTo, emailCc, and emailMetadata fields  
    - Potential Failures: Missing input fields, invalid email addresses, code execution errors.

#### 2.6 Email Dispatch

- **Overview:** Sends the composed alert email to HR using Gmail OAuth2 credentials.
- **Nodes Involved:**  
  - Send email to hr

- **Node Details:**

  - **Send email to hr**  
    - Type: Gmail node  
    - Role: Sends the email composed in the previous node to a fixed HR email address.  
    - Configuration:  
      - To: jyothi.swarup@techdome.net.in (hardcoded recipient)  
      - Subject: From input JSON’s `emailSubject`  
      - Message: From input JSON’s `emailBody`  
    - Credentials: Gmail OAuth2 for authenticated sending  
    - Input: JSON with email content and metadata  
    - Output: Email send status  
    - Potential Failures: Gmail API auth errors, rate limits, invalid email formatting.

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                          | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                   |
|-------------------------|----------------------------------------|----------------------------------------|-----------------------|-----------------------|------------------------------------------------------------------------------------------------|
| Trigger for new resume   | Google Drive Trigger                    | Detects new resume files in Drive      | None                  | Download resume       | Starts the workflow when a new resume file is added (e.g., to storage or inbox).              |
| Download resume          | Google Drive                           | Downloads detected resume file          | Trigger for new resume | Extract text          | Fetches the resume file from the source and makes it available for processing.                |
| Extract text             | Extract From File                      | Extracts text from the downloaded PDF  | Download resume        | Calculate avg span    | Pulls readable text from the downloaded resume using a PDF extraction step.                   |
| Azure OpenAI Chat Model  | Langchain Azure OpenAI Chat            | Sends text to GPT-4o-mini for analysis | (Indirect via Calculate avg span) | Calculate avg span    | Uses Azure OpenAI Chat to analyze or summarize the extracted resume content.                  |
| Structured Output Parser | Langchain Structured Output Parser     | Parses GPT output into structured JSON | Azure OpenAI Chat Model | Calculate avg span    |                                                                                                |
| Calculate avg span       | Langchain Agent                        | Computes average employment tenure     | Extract text, Azure OpenAI Chat Model, Structured Output Parser | Logic                 |                                                                                                |
| Logic                   | If node                               | Checks if average tenure < 12 months   | Calculate avg span     | Create email          | Applies conditional checks and routing (true/false) based on parsed results.                  |
| Create email             | Code node (JavaScript)                  | Builds personalized email alerts       | Logic                  | Send email to hr      | Generates a tailored email draft to the candidate or HR using the parsed data.                |
| Send email to hr         | Gmail                                | Sends alert email to HR                 | Create email           | None                  | Sends the composed message to HR via the configured email service.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node:**  
   - Type: Google Drive Trigger  
   - Set event to `fileCreated`  
   - Configure to trigger on a specific folder by Folder ID: `1KyX5RGqeF7v0sAvvaoBnJPTQoq4aOHLz`  
   - Set polling interval to every minute  
   - Attach valid Google Drive OAuth2 credentials  

2. **Add Google Drive Node (Download):**  
   - Type: Google Drive  
   - Operation: Download  
   - Set File ID to `={{ $json.webViewLink }}` from trigger output  
   - Attach the same Google Drive OAuth2 credentials  
   - Connect the trigger node output to this node input  

3. **Add Extract From File Node:**  
   - Type: Extract From File  
   - Operation: pdf  
   - Input: Binary data from "Download resume" node  
   - Connect "Download resume" output to this node  

4. **Add Langchain Azure OpenAI Chat Model Node:**  
   - Type: Langchain Azure OpenAI Chat  
   - Model: `gpt-4o-mini`  
   - Credentials: Azure Open AI API credentials (configured in n8n)  
   - Input: Resume text from "Extract text" node (pass as parameter)  

5. **Add Langchain Structured Output Parser Node:**  
   - Type: Langchain Structured Output Parser  
   - Configure JSON schema example to `{ "average": "18" }`  
   - Connect output of Azure OpenAI Chat model to this parser node  

6. **Add Langchain Agent Node (Calculate avg span):**  
   - Type: Langchain Agent  
   - Configure system message with detailed instructions for extracting employment experiences and calculating average tenure in months as per the given prompt (see overview section 2.3)  
   - Enable output parser  
   - Connect Extract Text node output as text input  
   - Connect AI model and output parser nodes accordingly (see connections)  

7. **Add If Node (Logic):**  
   - Type: If node  
   - Condition: Check if numeric value of `{{ $json.output.average }}` is less than 12  
   - Connect output of "Calculate avg span" to this node  

8. **Add Code Node (Create email):**  
   - Type: Code node (JavaScript)  
   - Paste the provided JavaScript code that formats email subject, body, recipients, and metadata based on input JSON fields  
   - Connect the true branch of the If node (Logic) to this node  

9. **Add Gmail Node (Send email to hr):**  
   - Type: Gmail  
   - Set recipient email to `jyothi.swarup@techdome.net.in`  
   - Subject: `={{ $json.emailSubject }}`  
   - Message: `={{ $json.emailBody }}`  
   - Attach Gmail OAuth2 credentials  
   - Connect "Create email" node output to this node  

10. **Add Sticky Notes (Optional for clarity):**  
    - Add notes near each node describing its function as per the sticky notes content in the workflow.

11. **Save and activate workflow:**  
    - Ensure all credentials are valid and permissions are granted for Google Drive and Gmail.  
    - Test trigger by uploading a resume file to the monitored Google Drive folder.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow integrates Azure OpenAI GPT-4o-mini via Langchain nodes in n8n for advanced text analysis.  | Refer to Azure OpenAI API documentation and Langchain n8n node docs for credential and usage setup.       |
| The resume folder ID and email recipient are hardcoded; customize these for your environment.             | Google Drive folder: `1KyX5RGqeF7v0sAvvaoBnJPTQoq4aOHLz`; Email: `jyothi.swarup@techdome.net.in`          |
| Email content generation includes risk classification and suggested HR actions, facilitating retention.  | See detailed JavaScript code in the "Create email" node for customization of messaging and metadata.      |
| PDF extraction quality depends on resume format; consider fallback or OCR nodes if PDFs are scanned images.| PDF extraction node limitations may require enhancements for non-text PDFs.                              |
| The workflow assumes employees with average tenure under 12 months are at high attrition risk.            | Threshold can be adjusted in the Logic (If) node as per HR policies.                                      |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.