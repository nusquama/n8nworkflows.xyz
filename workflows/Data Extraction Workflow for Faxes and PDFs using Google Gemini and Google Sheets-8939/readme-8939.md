Data Extraction Workflow for Faxes and PDFs using Google Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/data-extraction-workflow-for-faxes-and-pdfs-using-google-gemini-and-google-sheets-8939


# Data Extraction Workflow for Faxes and PDFs using Google Gemini and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of structured data from faxed or PDF documents, specifically targeting medical or administrative forms. It accepts a fax PDF upload via a web form, stores the file in Google Drive, uses Google Gemini (Google’s generative AI with PDF capabilities) to analyze the document content, processes and structures the extracted text into a strict JSON format, and finally appends the cleaned data into a Google Sheet for further use.

Logical blocks are grouped as follows:

- **1.1 Input Reception**: Receiving fax PDF uploads through a form and saving them to Google Drive.
- **1.2 File Preparation and AI Extraction**: Downloading the file, extracting binary data, and sending it to Google Gemini API for content extraction.
- **1.3 Text Processing and Structured Parsing**: Cleaning raw AI output, using a second AI call to convert text into strict JSON, and parsing structured output.
- **1.4 Data Storage**: Appending the structured data into a Google Sheet.
- **1.5 Configuration and Customization**: Nodes and sticky notes providing prompt customization, credential setup, and field mapping instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block provides an entry point for users to upload fax PDF files through a web form and automatically uploads the file to a designated Google Drive folder.

**Nodes Involved:**  
- On form submission  
- Upload file  
- Sticky Note (Start Here)  
- Sticky Note1 (Google Drive configuration)

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Starts workflow on user upload of fax PDF via web form.  
  - *Configuration:* Single required file upload field labeled “Upload File”.  
  - *Connections:* Outputs to “Upload file”.  
  - *Failure modes:* Form not submitted, file missing or corrupted upload, webhook misconfiguration.  
  - *Sticky Note:* Explains this is the workflow’s entry point and form setup.

- **Upload file**  
  - *Type:* Google Drive node (upload operation)  
  - *Role:* Uploads the received PDF into a specified Google Drive folder.  
  - *Configuration:*  
    - Drive: "My Drive"  
    - Folder ID: User must specify their Google Drive folder ID  
    - Input data field: “Upload_File” from form  
  - *Connections:* Receives input from “On form submission”, outputs to “Define Prompt”.  
  - *Credentials:* Google Drive OAuth2 credentials required.  
  - *Failure modes:* Authentication errors, folder permission issues, file size limits.  
  - *Sticky Note1:* Instructions on selecting credentials and specifying folder ID.

---

#### 2.2 File Preparation and AI Extraction

**Overview:**  
This block downloads the uploaded file from Google Drive, converts its binary data to a usable property, and sends it along with a customized prompt to Google Gemini’s PDF-capable generative model for extracting document content.

**Nodes Involved:**  
- Define Prompt  
- Google Drive (download)  
- Extract from File  
- Call Gemini 2.0 Flash with PDF Capabilities  
- Sticky Note2 (Prompt customization part 1)  
- Sticky Note3 (Google Gemini Call configuration)

**Node Details:**  

- **Define Prompt**  
  - *Type:* Set node  
  - *Role:* Defines the AI prompt instructing the model how to analyze the fax document.  
  - *Configuration:*  
    - A detailed prompt string specifying extraction rules for medical/admin forms, fields to extract, instructions to ignore irrelevant text, and focus on exact extraction without interpretation.  
  - *Connections:* Receives input from “Upload file”, outputs to “Google Drive” (download).  
  - *Failure modes:* Expression errors if prompt is malformed.  
  - *Sticky Note2:* Advises that the prompt can be edited to customize extraction.

- **Google Drive (download)**  
  - *Type:* Google Drive node (download operation)  
  - *Role:* Downloads the uploaded fax PDF by referencing file ID from “Upload file” node output.  
  - *Configuration:* File ID dynamically set from the “Upload file” node output JSON data.  
  - *Connections:* Input from “Define Prompt”, outputs to “Extract from File”.  
  - *Credentials:* Same Google Drive OAuth2 credentials as upload.  
  - *Failure modes:* File not found, permission denied, invalid file ID.

- **Extract from File**  
  - *Type:* Extract From File node  
  - *Role:* Converts binary file data (PDF) to a string property suitable for API consumption.  
  - *Configuration:* Operation set to “binaryToPropery”.  
  - *Connections:* Input from “Google Drive (download)”, outputs to “Call Gemini 2.0 Flash with PDF Capabilities”.  
  - *Failure modes:* Binary data missing or corrupted, unsupported file type.

- **Call Gemini 2.0 Flash with PDF Capabilities**  
  - *Type:* HTTP Request node  
  - *Role:* Sends the extracted PDF data and prompt to Google Gemini API to generate content.  
  - *Configuration:*  
    - POST request to Gemini 2.0 Flash model endpoint.  
    - JSON body includes inline PDF data (base64 from extract node) and prompt text from “Define Prompt”.  
    - Authentication via predefined Google Palm API credentials.  
  - *Connections:* Input from “Extract from File”, outputs to “Code” node.  
  - *Credentials:* Google Palm API credentials required.  
  - *Failure modes:* API quota exceeded, invalid credentials, malformed request.  
  - *Sticky Note3:* Highlights required credential setup.

---

#### 2.3 Text Processing and Structured Parsing

**Overview:**  
This block processes the raw AI output text, cleans it, and makes a second AI call to convert the extracted text into a strict JSON format matching a predefined schema. The output is parsed for subsequent storage.

**Nodes Involved:**  
- Code  
- Google Gemini Chat Model  
- Structured Output Parser  
- Basic LLM Chain  
- Sticky Note4 (AI Brain part 2 customization)  
- Sticky Note6 (Extraction values and field mapping)

**Node Details:**  

- **Code**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Cleans the raw output from Gemini 2.0 Flash API by extracting candidate text parts, removing markdown syntax like bold or list markers, and trimming whitespace.  
  - *Key expressions:* References $json.candidates and content parts to extract text.  
  - *Connections:* Input from “Call Gemini 2.0 Flash with PDF Capabilities”, outputs to “Google Gemini Chat Model”.  
  - *Failure modes:* Missing or unexpected JSON structure, undefined properties.

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini Chat node  
  - *Role:* Sends cleaned text to Google Gemini Chat API to reformat and structure it as JSON according to a strict schema.  
  - *Connections:* Input from “Code”, outputs to “Structured Output Parser”.  
  - *Credentials:* Google Palm API credentials required.  
  - *Failure modes:* API errors, invalid credentials, network issues.

- **Structured Output Parser**  
  - *Type:* LangChain output parser node  
  - *Role:* Parses the AI response to ensure it adheres to a JSON schema example describing key patient and medical fields.  
  - *Configuration:* JSON example schema includes fields like PatientID, SOCDate, MedicalRecord, ProviderNo, PatientName, PatientAddress, DateOfBirth, SafetyMeasures, Allergies, Gender, Mental Status, Prognosis, DateSigned.  
  - *Connections:* Input from “Google Gemini Chat Model”, outputs to “Basic LLM Chain”.  
  - *Failure modes:* Parsing errors if output is not valid JSON.

- **Basic LLM Chain**  
  - *Type:* LangChain LLM Chain node  
  - *Role:* Enforces strict JSON parsing rules and outputs a clean JSON object with all required keys, even if null.  
  - *Configuration:*  
    - Prompt instructs to return only valid JSON matching the schema, no markdown or extra text.  
  - *Connections:* Input from “Structured Output Parser”, outputs to “Append row in sheet”.  
  - *Failure modes:* Parsing failures, prompt misconfiguration.  
  - *Sticky Note4:* Instructions on setting credentials and customizing output fields.  
  - *Sticky Note6:* Advises matching fields in Google Sheets and LLM Chain prompt.

---

#### 2.4 Data Storage

**Overview:**  
The final block appends the structured data extracted from the fax into a Google Sheet for record-keeping or further processing.

**Nodes Involved:**  
- Append row in sheet  
- Sticky Note5 (Google Sheets configuration)

**Node Details:**  

- **Append row in sheet**  
  - *Type:* Google Sheets node (append operation)  
  - *Role:* Appends a new row with extracted patient and medical data into a specified Google Sheet and sheet tab.  
  - *Configuration:*  
    - Document ID: User’s Google Sheet ID  
    - Sheet name: Typically “Sheet1” (gid=0)  
    - Columns mapped explicitly from extracted JSON keys like Patient ID, SOC Date, Medical Record, Provider No., Patient Name, Patient Address, Date of Birth, Safety Measures, Allergies, Gender, Mental Status, Prognosis, Date Signed.  
  - *Connections:* Input from “Basic LLM Chain”.  
  - *Credentials:* Google Sheets OAuth2 credentials required.  
  - *Failure modes:* Authentication errors, sheet not found, column mismatch, quota exceeded.  
  - *Sticky Note5:* Instructions for configuring credentials, document ID, and column headers.

---

### 3. Summary Table

| Node Name                           | Node Type                     | Functional Role                               | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                  |
|-----------------------------------|-------------------------------|-----------------------------------------------|-------------------------------|----------------------------------------|--------------------------------------------------------------|
| On form submission                | Form Trigger                  | Entry point; receives fax PDF upload           | —                             | Upload file                            | ## 1. Start Here: Form Trigger \n* This is the entry point for the workflow.\n* It creates a web form where a user can upload the FAX PDF file. |
| Upload file                      | Google Drive (upload)         | Uploads PDF to Google Drive                     | On form submission            | Define Prompt                         | ## 2. Configure: Upload to Google Drive\n* Select your Google Drive account from the 'Credentials' dropdown.\n* Change the Folder ID to the specific Google Drive folder where you want to save the faxes. |
| Define Prompt                   | Set                           | Defines AI prompt for Gemini extraction         | Upload file                   | Google Drive (download)               | ## 3. Customize AI Brain (Part 1)\n* You can edit the *prompt* value here to change the extraction rules or ask for different information. |
| Google Drive                   | Google Drive (download)       | Downloads uploaded PDF file                      | Define Prompt                 | Extract from File                    |                                                              |
| Extract from File               | Extract from File             | Converts binary file data to property           | Google Drive                  | Call Gemini 2.0 Flash with PDF Capabilities |                                                              |
| Call Gemini 2.0 Flash with PDF Capabilities | HTTP Request                 | Sends PDF and prompt to Google Gemini API        | Extract from File             | Code                                | ## 4. First AI Call: PDF Extraction\n* **ACTION REQUIRED**: You must select your "Google Palm API" credentials from the dropdown for this node to work. |
| Code                            | Code                          | Cleans AI output text                            | Call Gemini 2.0 Flash with PDF Capabilities | Google Gemini Chat Model            |                                                              |
| Google Gemini Chat Model        | LangChain Google Gemini Chat  | Second AI call to structure text to strict JSON | Code                         | Structured Output Parser             | ## 5. Customize AI Brain (Part 2)\nThis second AI call structures the cleaned text into a strict JSON format.\n* **ACTION REQUIRED**: In the connected Google Gemini Chat Model node, select your Gemini API credentials.\n* To change the final output fields, edit the JSON schema inside the Prompt field of this node. |
| Structured Output Parser        | LangChain Output Parser       | Parses AI JSON output to ensure schema compliance | Google Gemini Chat Model     | Basic LLM Chain                     |                                                              |
| Basic LLM Chain                | LangChain LLM Chain           | Enforces strict JSON format and outputs clean JSON | Structured Output Parser     | Append row in sheet                 | ## Extraction Values\n* Once the workflow is set up and all nodes are configured, define the fields you want to extract from the *fax* content.\n\n* Add corresponding column fields in *Google Sheets* for each data point that needs to be extracted.\n\n* In the *LLM Chain* node prompt, specify the same fields as keys and provide example values to guide the extraction.\n\n* Update the *Structured Output Parser* with these keys and expected values to ensure the output is consistently structured and aligned with your Google Sheets columns. |
| Append row in sheet            | Google Sheets (append)        | Appends extracted data as a new row in Google Sheet | Basic LLM Chain             | —                                  | ## Final Step: Save to Google Sheets\n* **ACTION REQUIRED**: Configure your Google Sheets credentials.\n* **ACTION REQUIRED**: Update the Document ID with the ID of your own Google Sheet.\n* Ensure the column names in this node match the header row in your target Google Sheet. |
| Sticky Note                    | Sticky Note                  | Provides user guidance                          | —                             | —                                  | See individual sticky notes above.                            |
| Sticky Note1                   | Sticky Note                  | Guidance on Google Drive upload configuration   | —                             | —                                  | See above.                                                    |
| Sticky Note2                   | Sticky Note                  | Guidance on prompt customization                  | —                             | —                                  | See above.                                                    |
| Sticky Note3                   | Sticky Note                  | Guidance on Gemini API credential setup          | —                             | —                                  | See above.                                                    |
| Sticky Note4                   | Sticky Note                  | Guidance on second AI call and JSON schema       | —                             | —                                  | See above.                                                    |
| Sticky Note5                   | Sticky Note                  | Guidance on Google Sheets configuration          | —                             | —                                  | See above.                                                    |
| Sticky Note6                   | Sticky Note                  | Guidance on extraction values and field mapping  | —                             | —                                  | See above.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission")**  
   - Add a new Form Trigger node.  
   - Configure with a form titled “Fax File”.  
   - Add a required file upload field labeled “Upload File”.  
   - This node will serve as the webhook entry point.

2. **Add a Google Drive node ("Upload file") - Upload operation**  
   - Connect it to the Form Trigger node.  
   - Configure to upload the received file from the form field “Upload_File”.  
   - Select your Google Drive OAuth2 credentials.  
   - Set “Drive” to “My Drive”.  
   - Set “Folder ID” to your target Google Drive folder ID where faxes will be saved.

3. **Add a Set node ("Define Prompt")**  
   - Connect it downstream from the “Upload file” node.  
   - Create a string field named “prompt”.  
   - Paste the detailed prompt text guiding the AI on how to analyze the fax (medical form extraction instructions).  
   - This defines the AI’s extraction rules.

4. **Add a Google Drive node ("Google Drive") - Download operation**  
   - Connect from “Define Prompt”.  
   - Configure to download the file using the file ID from the “Upload file” node’s output (use an expression to dynamically reference this ID).  
   - Use the same Google Drive credentials as before.

5. **Add an Extract From File node ("Extract from File")**  
   - Connect from “Google Drive” (download).  
   - Set operation to “binaryToProperty” to convert binary PDF data into a string property.

6. **Add an HTTP Request node ("Call Gemini 2.0 Flash with PDF Capabilities")**  
   - Connect from “Extract from File”.  
   - Configure:  
     - Method: POST  
     - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent`  
     - Authentication: Select your Google Palm API credentials.  
     - Body type: JSON  
     - Body content: JSON object with “contents” array, including inline PDF data (base64 encoded from extracted file data) and the prompt text from “Define Prompt”. Use expressions to insert these dynamically.

7. **Add a Code node ("Code")**  
   - Connect from “Call Gemini 2.0 Flash with PDF Capabilities”.  
   - Paste the JavaScript code that extracts the first candidate’s text content, removes markdown artifacts (e.g., bolds and list markers), and trims whitespace.  
   - This cleans the raw AI response.

8. **Add a LangChain Google Gemini Chat node ("Google Gemini Chat Model")**  
   - Connect from the Code node.  
   - Select Google Palm API credentials.  
   - This node sends the cleaned text for a second AI call to produce structured JSON.

9. **Add a LangChain Output Parser node ("Structured Output Parser")**  
   - Connect from “Google Gemini Chat Model”.  
   - Paste the JSON schema example defining the structure expected from the AI, including keys like PatientID, SOCDate, MedicalRecord, ProviderNo, PatientName, PatientAddress, DateOfBirth, SafetyMeasures, Allergies, Gender, Mental Status, Prognosis, DateSigned.

10. **Add a LangChain LLM Chain node ("Basic LLM Chain")**  
    - Connect from “Structured Output Parser”.  
    - Configure the prompt to enforce strict JSON output matching the schema, with instructions to return only valid JSON without markdown or extra text.  
    - Enable output parsing.  
    - Enable “retry on fail” for robustness.

11. **Add a Google Sheets node ("Append row in sheet")**  
    - Connect from “Basic LLM Chain”.  
    - Select Google Sheets OAuth2 credentials.  
    - Configure the node to append rows to your target Google Sheet document ID and sheet name (e.g., “Sheet1”).  
    - Map columns explicitly to JSON keys returned by the AI: Patient ID, SOC Date, Medical Record, Provider No., Patient Name, Patient Address, Date of Birth, Safety Measures, Allergies, Gender, Mental Status, Prognosis, Date Signed.  
    - Ensure headers in your Google Sheet exactly match these field names.

12. **Set Sticky Notes**  
    - Add sticky notes near nodes with instructions for configuration and customization as per the original workflow. This helps maintain clarity for users.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini API for advanced document content extraction with PDF support.                        | Official Google Gemini API documentation and developer guides.                                        |
| Google Drive and Google Sheets OAuth2 credentials must be configured with appropriate scopes for file and sheet access. | Ensure OAuth2 consent and API access are correctly set up in Google Cloud Console.                     |
| The detailed prompt can be customized to adjust extraction focus or add additional fields if needed.                  | Adjust prompt in “Define Prompt” node.                                                                |
| For best results, ensure the Google Sheet column headers exactly match the keys defined in the JSON schema and LLM prompt. | Prevents errors in appending rows and maintains data integrity.                                       |
| Sticky notes in the workflow provide essential configuration and operational tips.                                    | Use as inline documentation within n8n editor.                                                        |
| The workflow is designed to handle medical fax forms but can be adapted to other structured document types by changing the prompt and schema. | Flexibility for other use cases by modifying AI instructions and output parsing.                       |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.