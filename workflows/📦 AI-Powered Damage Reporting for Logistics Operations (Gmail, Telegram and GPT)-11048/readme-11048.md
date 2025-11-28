📦 AI-Powered Damage Reporting for Logistics Operations (Gmail, Telegram and GPT)

https://n8nworkflows.xyz/workflows/---ai-powered-damage-reporting-for-logistics-operations--gmail--telegram-and-gpt--11048


# 📦 AI-Powered Damage Reporting for Logistics Operations (Gmail, Telegram and GPT)

### 1. Workflow Overview

This workflow automates damage reporting in logistics operations using AI-powered image analysis and barcode reading integrated with Telegram, OpenAI GPT models, and Gmail. Warehouse operators interact via Telegram to submit photos of damaged pallets and their barcodes. The workflow processes these images, generates a structured damage report, extracts pallet identification, and emails a formatted report to the quality control team. It is designed to streamline and automate damage documentation without manual report writing.

**Logical Blocks:**

- **1.1 Input Reception and Validation:** Receives Telegram messages, checks if they contain images, and guides the operator accordingly.
- **1.2 State Management:** Loads and stores workflow-global state variables to track report generation progress.
- **1.3 AI Image Analysis:** Uses GPT-4o to analyze damage images and GPT-4o Mini to read barcode images.
- **1.4 Report Generation and Storage:** Formats the AI output into an HTML email report and stores report data.
- **1.5 Report Distribution and Confirmation:** Sends the email report to the quality team and confirms successful submission to the operator.
- **1.6 Initialization:** Resets workflow state variables after report completion.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Validation

**Overview:**  
This block listens for Telegram messages, verifies if the message contains an image, and responds with instructions if not.

**Nodes Involved:**  
- Telegram Trigger  
- Image Received ? (If)  
- Instructions to Operator

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point triggered by incoming Telegram messages containing updates of type "message".  
  - *Config:* Monitors all incoming messages on the configured Telegram bot.  
  - *Input:* Incoming Telegram updates.  
  - *Output:* Passes message data downstream.  
  - *Edge Cases:* Failure if Telegram credentials are invalid or webhook misconfigured.

- **Image Received ?**  
  - *Type:* If  
  - *Role:* Checks if the incoming Telegram message contains a photo array with at least one file_id.  
  - *Config:* Condition tests existence of `message.photo[0].file_id`.  
  - *Input:* Output from Telegram Trigger.  
  - *Output:* Two branches — true if image exists, false otherwise.  
  - *Edge Cases:* May fail if message structure differs or photo array missing.

- **Instructions to Operator**  
  - *Type:* Telegram  
  - *Role:* Sends instructions to the operator if no image is detected, guiding on the required photo submissions.  
  - *Config:* Sends a multi-step instruction text asking first for a damaged pallet photo, then the barcode photo, with force reply enabled for better UX.  
  - *Input:* False branch from Image Received ?.  
  - *Output:* Sends message to Telegram chat corresponding to the triggering user.  
  - *Edge Cases:* Telegram API failures, message send errors.

---

#### 1.2 State Management

**Overview:**  
Manages global static data to track whether a report has been generated and stores report and barcode information temporarily.

**Nodes Involved:**  
- Load Report State  
- Report Generated (If)  
- Load Report Infos  
- Store Report  
- Store Barcode  
- Reinitialise State Variables

**Node Details:**

- **Load Report State**  
  - *Type:* Code  
  - *Role:* Reads global static data (stored in workflow static memory) to check if a damage report exists.  
  - *Config:* Returns `report_bool` flag and `report` text (or defaults).  
  - *Input:* True branch from Image Received ?.  
  - *Output:* JSON with report state info.  
  - *Edge Cases:* Static data uninitialized or corrupted.

- **Report Generated**  
  - *Type:* If  
  - *Role:* Tests if a report has already been generated (`report_bool == true`).  
  - *Config:* Boolean check on `report_bool`.  
  - *Input:* Output of Load Report State.  
  - *Output:* Branches depending on report existence.  
  - *Edge Cases:* False negatives if state not properly updated.

- **Load Report Infos**  
  - *Type:* Code  
  - *Role:* Loads report data and barcode from static global data for generating final email report.  
  - *Config:* Returns `report_bool`, `report`, and `barcode` variables.  
  - *Input:* After barcode is stored.  
  - *Output:* JSON with full report info.  
  - *Edge Cases:* Missing or stale data.

- **Store Report**  
  - *Type:* Code  
  - *Role:* Stores the damage report text returned by AI into global static data and sets the report flag true.  
  - *Config:* Uses first input JSON `output[0].content[0].text` as report text.  
  - *Input:* AI Damage Analysis output.  
  - *Output:* Updated report state.  
  - *Edge Cases:* AI node outputs unexpected format.

- **Store Barcode**  
  - *Type:* Code  
  - *Role:* Stores the barcode text extracted by AI into global static data.  
  - *Config:* Stores barcode from input JSON path `['0'].content[0].text`.  
  - *Input:* Output of Extract Barcode Value.  
  - *Output:* Updated barcode state.  
  - *Edge Cases:* AI output format errors.

- **Reinitialise State Variables**  
  - *Type:* Code  
  - *Role:* Resets static data for `report_bool` and `report` to initial values, preparing for new report.  
  - *Config:* Sets `report_bool` false and `report` to "Not available".  
  - *Input:* After report sent confirmation.  
  - *Output:* Reset state.  
  - *Edge Cases:* None significant, but should be idempotent.

---

#### 1.3 AI Image Analysis

**Overview:**  
Processes images through OpenAI GPT models: one for damage analysis of the pallet photo, another for reading the barcode image text.

**Nodes Involved:**  
- Download Pallet Image  
- AI Damage Analysis (GPT-4o)  
- Download Barcode Image  
- AI Barcode Reader (GPT-4o Mini)  
- Extract Barcode Value  
- Request Barcode Photo

**Node Details:**

- **Download Pallet Image**  
  - *Type:* Telegram (File Download)  
  - *Role:* Downloads the fourth photo from the Telegram message (assumed to be the pallet damage image).  
  - *Config:* Uses `message.photo[3].file_id` from Telegram Trigger.  
  - *Input:* True branch of Report Generated.  
  - *Output:* Image file (base64) for AI analysis.  
  - *Edge Cases:* Index errors if photo array length <4, download failures.

- **AI Damage Analysis (GPT-4o)**  
  - *Type:* OpenAI (LangChain)  
  - *Role:* Analyzes the damage image and generates structured damage report text.  
  - *Config:* Uses GPT-4o model with instructions enforcing output formatting (Damage Summary, Observed Damage, Severity, Recommended Actions).  
  - *Input:* Base64 image from Download Pallet Image.  
  - *Output:* Structured text report.  
  - *Edge Cases:* Model timeouts, hallucination risks mitigated by instructions.

- **Request Barcode Photo**  
  - *Type:* Telegram  
  - *Role:* Sends a prompt to the operator asking for a photo of the pallet barcode after damage report creation.  
  - *Config:* Text asks specifically for a clear pallet barcode photo with force reply enabled.  
  - *Input:* After AI Damage Analysis.  
  - *Output:* Telegram message to operator.  
  - *Edge Cases:* Telegram API errors.

- **Download Barcode Image**  
  - *Type:* Telegram (File Download)  
  - *Role:* Downloads the fourth photo from the Telegram message (assumed to be the barcode image).  
  - *Config:* Uses `message.photo[3].file_id`.  
  - *Input:* False branch of Report Generated (when report is not already generated).  
  - *Output:* Base64 image of barcode for AI reading.  
  - *Edge Cases:* Same as Download Pallet Image.

- **AI Barcode Reader (GPT-4o Mini)**  
  - *Type:* OpenAI (LangChain)  
  - *Role:* Reads the barcode text from the image and outputs just the barcode value.  
  - *Config:* Uses GPT-4o Mini model with simple output instructions "Read the barcode, just output the value".  
  - *Input:* Base64 image from Download Barcode Image.  
  - *Output:* Barcode string.  
  - *Edge Cases:* OCR failures, unclear images.

- **Extract Barcode Value**  
  - *Type:* Set  
  - *Role:* Extracts the barcode string from AI output JSON for storage.  
  - *Config:* Assigns `['0'].content[0].text` from AI output to a variable.  
  - *Input:* Output of AI Barcode Reader.  
  - *Output:* Clean barcode string JSON.  
  - *Edge Cases:* Unexpected AI output format.

---

#### 1.4 Report Generation and Storage

**Overview:**  
Generates a visually styled HTML report combining barcode and damage report, preparing it for email delivery.

**Nodes Involved:**  
- Load Report Infos  
- Generate Report

**Node Details:**

- **Load Report Infos** (also part of Block 1.2)  
  - *Role:* Loads current report and barcode info for email generation.  
  - *Input:* After storing barcode.  
  - *Output:* JSON with report_bool, report, barcode.

- **Generate Report**  
  - *Type:* Code  
  - *Role:* Creates an HTML email body with branding (LogiGreen logo, colors), formats text with line breaks, and sets email subject dynamically with pallet number.  
  - *Config:* Uses embedded CSS styling, includes footer with company link, formats the damage report text replacing newlines with `<br>`.  
  - *Input:* JSON with report data.  
  - *Output:* JSON with `email_title` and `email_html` for email node.  
  - *Edge Cases:* HTML injection if report text is malformed (unlikely since AI output is controlled).

---

#### 1.5 Report Distribution and Confirmation

**Overview:**  
Sends the generated report via Gmail and confirms success to the operator via Telegram.

**Nodes Involved:**  
- Send Report by Email  
- Confirm Report Sent to Operator  
- Reinitialise State Variables (also in Block 1.2)

**Node Details:**

- **Send Report by Email**  
  - *Type:* Gmail  
  - *Role:* Sends the HTML damage report email to the quality team email address.  
  - *Config:* Uses Gmail OAuth2 credentials, subject and message are dynamically set from "Generate Report" output.  
  - *Input:* Output from Generate Report.  
  - *Output:* Email sent confirmation.  
  - *Edge Cases:* SMTP or OAuth2 auth failures, invalid email address placeholder `<ADD_EMAIL_QUALITY_TEAM>` must be replaced.

- **Confirm Report Sent to Operator**  
  - *Type:* Telegram  
  - *Role:* Sends confirmation message back to the Telegram operator indicating successful report creation and email dispatch.  
  - *Config:* Fixed confirmation text with branding, sent to the triggering chat ID.  
  - *Input:* After successful email send.  
  - *Output:* Telegram message sent.  
  - *Edge Cases:* Telegram API errors.

- **Reinitialise State Variables**  
  - *Role:* Resets report state for next report cycle. (See Block 1.2)

---

#### 1.6 Initialization (One-time Setup)

**Overview:**  
Initializes global state variables on first workflow run or after completion to enable proper tracking of report generation.

**Nodes Involved:**  
- Reinitialise State Variables (also in Block 1.2)

**Node Details:**

- As above; recommended to run once manually or triggered after completion.

---

### 3. Summary Table

| Node Name                | Node Type                    | Functional Role                             | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                   |
|--------------------------|------------------------------|---------------------------------------------|----------------------------|----------------------------|-----------------------------------------------------------------------------------------------|
| Telegram Trigger          | Telegram Trigger             | Entry point for Telegram message updates    | —                          | Image Received ?            | Covered by Sticky Note8: "1. Telegram Trigger and message format check"                      |
| Image Received ?          | If                          | Checks if message contains a photo          | Telegram Trigger            | Load Report State (true), Instructions to Operator (false) | Sticky Note9: "2. If the message is not an image, send instructions"                         |
| Instructions to Operator  | Telegram                    | Sends instructions if no image received     | Image Received ? (false)    | —                          | Sticky Note9                                                                                   |
| Load Report State         | Code                        | Loads report state from global static data  | Image Received ? (true)     | Report Generated            | Sticky Note10: "3. Load state variables and check whether a report was already generated"    |
| Report Generated          | If                          | Branches based on whether report exists     | Load Report State           | Download Barcode Image (false), Download Pallet Image (true) | Sticky Note10                                                                                  |
| Download Barcode Image    | Telegram (File Download)     | Downloads barcode photo                      | Report Generated (false)    | AI Barcode Reader (GPT-4o Mini) | Sticky Note13: "4. Download the photo to extract the bar code"                              |
| AI Barcode Reader (GPT-4o Mini) | OpenAI (LangChain)           | Reads barcode text from image                | Download Barcode Image      | Extract Barcode Value       | Sticky Note13                                                                                  |
| Extract Barcode Value     | Set                         | Extracts barcode string from AI output      | AI Barcode Reader           | Store Barcode               | Sticky Note13                                                                                  |
| Store Barcode             | Code                        | Stores barcode in global static data        | Extract Barcode Value       | Load Report Infos           | Sticky Note15: "6. Load the state variables used for the report"                            |
| Load Report Infos         | Code                        | Loads report and barcode info                | Store Barcode               | Generate Report             | Sticky Note15                                                                                  |
| Download Pallet Image     | Telegram (File Download)     | Downloads damage photo                       | Report Generated (true)     | AI Damage Analysis (GPT-4o) | Sticky Note12: "5. Download the image, generate the report , store it, and request the barcode" |
| AI Damage Analysis (GPT-4o) | OpenAI (LangChain)           | Analyzes damage photo and generates report  | Download Pallet Image       | Request Barcode Photo, Store Report | Sticky Note12                                                                           |
| Request Barcode Photo     | Telegram                    | Requests barcode photo from operator        | AI Damage Analysis          | —                          | Sticky Note12                                                                                  |
| Store Report             | Code                        | Stores damage report in global static data  | AI Damage Analysis          | —                          | Sticky Note12                                                                                  |
| Generate Report           | Code                        | Creates HTML email report with branding     | Load Report Infos           | Send Report by Email        | Sticky Note14: "7. Generate the final report, send it by email, and confirm to the operator" |
| Send Report by Email      | Gmail                       | Sends email to quality team                  | Generate Report             | Confirm Report Sent to Operator | Sticky Note14                                                                               |
| Confirm Report Sent to Operator | Telegram                    | Confirms report sent to Telegram user       | Send Report by Email        | Reinitialise State Variables | Sticky Note14                                                                               |
| Reinitialise State Variables | Code                        | Resets global state for next report          | Confirm Report Sent to Operator | —                      |                                                                                               |
| Sticky Note               | Sticky Note                 | Documentation and overview                   | —                          | —                          | Contains detailed project overview and setup instructions                                   |
| Sticky Note1              | Sticky Note                 | YouTube tutorial link                         | —                          | —                          | Contains link: [Tutorial](https://www.youtube.com/watch?v=3Xdo1pzd8rw)                      |
| Sticky Note8              | Sticky Note                 | Section 1 header                              | —                          | —                          | "1. Telegram Trigger and message format check"                                              |
| Sticky Note9              | Sticky Note                 | Section 2 header                              | —                          | —                          | "2. If the message is not an image, send instructions"                                     |
| Sticky Note10             | Sticky Note                 | Section 3 header                              | —                          | —                          | "3. Load state variables and check whether a report was already generated"                  |
| Sticky Note12             | Sticky Note                 | Section 5 header                              | —                          | —                          | "5. Download the image, generate the report , store it, and request the barcode"            |
| Sticky Note13             | Sticky Note                 | Section 4 header                              | —                          | —                          | "4. Download the photo to extract the bar code"                                            |
| Sticky Note14             | Sticky Note                 | Section 7 header                              | —                          | —                          | "7. Generate the final report, send it by email, and confirm to the operator"               |
| Sticky Note15             | Sticky Note                 | Section 6 header                              | —                          | —                          | "6. Load the state variables used for the report"                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Monitor updates of type "message"  
   - Credentials: Connect to Telegram Bot API  
   - Position: Start node

2. **Add If Node "Image Received ?"**  
   - Check if `message.photo[0].file_id` exists (string exists condition)  
   - Input: Telegram Trigger output  
   - Output: True (has image), False (no image)

3. **Add Telegram Node "Instructions to Operator" (False branch)**  
   - Text: Multi-line instructions guiding operator to upload damaged pallet photo and barcode photo  
   - Chat ID: Use expression to get `message.chat.id` from Telegram Trigger  
   - Enable force reply (selective)  
   - Credentials: Telegram Bot API

4. **Add Code Node "Load Report State" (True branch)**  
   - JS Code:  
     ```js
     let workflowData = $getWorkflowStaticData('global');
     return [{ json: { report_bool: workflowData.report_bool || false, report: workflowData.report || "Not available" } }];
     ```  
   - Input: True output from "Image Received ?"

5. **Add If Node "Report Generated"**  
   - Condition: Boolean equals true on `report_bool` from Load Report State  
   - Input: Load Report State

6. **Add Telegram Node "Download Barcode Image" (False branch of Report Generated)**  
   - File ID: Use expression `{{$json.message.photo[3].file_id}}` (4th photo)  
   - Resource: file  
   - Credentials: Telegram Bot API

7. **Add OpenAI Node "AI Barcode Reader (GPT-4o Mini)"**  
   - Model: chatgpt-4o-latest (GPT-4o Mini)  
   - Operation: analyze image  
   - Input type: base64  
   - Text prompt: "Read the barcode, just output the value, nothing else."  
   - Input: Download Barcode Image output  
   - Credentials: OpenAI API Key

8. **Add Set Node "Extract Barcode Value"**  
   - Assign `['0'].content[0].text` to a variable (extract string from AI output)  
   - Input: AI Barcode Reader output

9. **Add Code Node "Store Barcode"**  
   - JS Code:  
     ```js
     let workflowStaticData = $getWorkflowStaticData('global');
     workflowStaticData.barcode = $input.first().json['0'].content[0].text;
     return { barcode: workflowStaticData.barcode };
     ```  
   - Input: Extract Barcode Value

10. **Add Code Node "Load Report Infos"**  
    - JS Code:  
      ```js
      let workflowData = $getWorkflowStaticData('global');
      return [{ json: { report_bool: workflowData.report_bool || false, report: workflowData.report || "Not available", barcode: workflowData.barcode || "Not available" } }];
      ```  
    - Input: Store Barcode

11. **Add Code Node "Generate Report"**  
    - JS Code: Generates HTML email body with embedded CSS, using `report`, `barcode`, and `report_bool` from input JSON.  
    - Input: Load Report Infos

12. **Add Gmail Node "Send Report by Email"**  
    - Send To: Replace `<ADD_EMAIL_QUALITY_TEAM>` with actual email address  
    - Subject: Use expression from Generate Report `email_title`  
    - Message: Use expression from Generate Report `email_html`  
    - Credentials: Gmail OAuth2

13. **Add Telegram Node "Confirm Report Sent to Operator"**  
    - Text: Confirmation message "✅ Your damage report has been generated and sent successfully..."  
    - Chat ID: Expression from Telegram Trigger `message.chat.id`  
    - Credentials: Telegram Bot API

14. **Add Code Node "Reinitialise State Variables"**  
    - JS Code:  
      ```js
      let workflowStaticData = $getWorkflowStaticData('global');
      workflowStaticData.report_bool = false;
      workflowStaticData.report = "Not available";
      return workflowStaticData;
      ```  
    - Input: Confirm Report Sent to Operator

15. **Add Telegram Node "Download Pallet Image" (True branch of Report Generated)**  
    - File ID: `{{$json.message.photo[3].file_id}}` (4th photo)  
    - Credentials: Telegram Bot API

16. **Add OpenAI Node "AI Damage Analysis (GPT-4o)"**  
    - Model: gpt-4o  
    - Operation: analyze image with base64 input  
    - Prompt: Detailed instructions to generate structured damage report in specified format without hallucination  
    - Credentials: OpenAI API Key

17. **Connect AI Damage Analysis outputs:**  
    - To Telegram Node "Request Barcode Photo"  
      - Text: Prompt user to send barcode photo with force reply  
      - Credentials: Telegram Bot API  
    - To Code Node "Store Report"  
      - JS Code:  
        ```js
        let workflowStaticData = $getWorkflowStaticData('global');
        workflowStaticData.report_bool = true;
        workflowStaticData.report = $input.first().json.output[0].content[0].text;
        return { report_bool: workflowStaticData.report_bool, report: workflowStaticData.report };
        ```

18. **Connect Store Report** to Store Barcode flow (step 10) after barcode extraction completes.

19. **Set up Sticky Notes** to document each block clearly as per the original workflow, including the project overview, instructions, and YouTube tutorial link.

20. **Test end-to-end:**  
    - Send damaged pallet photo via Telegram  
    - Verify AI damage report generation  
    - Send barcode photo in reply  
    - Verify barcode extraction  
    - Confirm email received by quality team  
    - Confirm Telegram confirmation message

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow designed for warehouse operators to generate damage reports via Telegram with AI assistance.                                    | Project overview node in workflow                |
| [YouTube Tutorial Video](https://www.youtube.com/watch?v=3Xdo1pzd8rw) linked in Sticky Note1.                                            | Tutorial link for visual walkthrough             |
| Branding uses LogiGreen logo and colors, customizable in "Generate Report" node HTML template.                                            | Branding customization instructions              |
| Requires Telegram Bot API credentials configured in n8n credentials manager.                                                             | Telegram integration                             |
| Requires OpenAI API key for GPT-4o and GPT-4o Mini models (Vision enabled).                                                               | OpenAI integration                               |
| Requires Gmail OAuth2 credentials and email recipient address configured in "Send Report by Email" node.                                  | Email delivery setup                             |
| AI prompts enforce strict output formatting and discourage hallucinations to ensure factual damage reports.                               | AI node configurations                           |
| Workflow uses n8n global static data to maintain state across multiple Telegram message events per user session.                         | State management strategy                         |
| Edge cases include Telegram message format deviations, AI model timeouts, image download failures, and auth errors with APIs.            | Operational considerations                        |
| The workflow assumes the pallet and barcode photos are sent as the 4th photo in the Telegram message; adjust indexing if needed.          | Important input data assumption                   |

---

**Disclaimer:** The text and workflow description originate exclusively from an automated n8n workflow. All data handled is legal and public, and content fully complies with content policies.