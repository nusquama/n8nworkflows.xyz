Analyze Sales Deal Outcomes with Azure OpenAI and Google Sheets Reporting

https://n8nworkflows.xyz/workflows/analyze-sales-deal-outcomes-with-azure-openai-and-google-sheets-reporting-8479


# Analyze Sales Deal Outcomes with Azure OpenAI and Google Sheets Reporting

### 1. Workflow Overview

This workflow is designed to analyze sales deal outcomes by leveraging Azure OpenAI's language models and automating reporting through Google Sheets and email digests. It targets sales and marketing teams who want to extract insights from notes and emails related to deals, process them with AI to generate structured feedback, store these insights in Google Sheets, and send summarized reports via email.

Logical blocks include:

- **1.1 Input Trigger & Data Retrieval:** Manual workflow trigger and fetching raw sales notes and emails from Google Sheets.
- **1.2 Data Extraction & Preparation:** Extracting relevant text fields (notes/emails) and batching for processing.
- **1.3 AI Processing:** Using Azure OpenAI chat models encapsulated in LangChain agents to analyze the extracted data and parse AI responses.
- **1.4 Data Storage:** Saving AI-processed feedback back to Google Sheets.
- **1.5 Reporting:** Filtering recent data and sending an email digest summary.
  
---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger & Data Retrieval

- **Overview:**  
  This block initiates the workflow manually and retrieves the raw sales data from a Google Sheets spreadsheet.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet (Google Sheets)  

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow when executed manually.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Triggers "Get row(s) in sheet" node.  
    - Failures: None typical, but manual execution required.
  
  - **Get row(s) in sheet**  
    - Type: Google Sheets (Read operation)  
    - Role: Retrieves rows from a specified Google Sheet containing sales notes and emails.  
    - Configuration: Setup with credential for Google Sheets OAuth2, configured with spreadsheet ID and sheet name/range.  
    - Key variables: Defines which rows to fetch, likely all or filtered.  
    - Inputs: Trigger from manual node.  
    - Outputs: Provides raw data to "Extract Notes/Emails".  
    - Failures: Authentication errors, sheet not found, empty data, API rate limits.

---

#### 1.2 Data Extraction & Preparation

- **Overview:**  
  Extracts notes and emails from the retrieved rows and splits data into batches suitable for AI processing.

- **Nodes Involved:**  
  - Extract Notes/Emails (Code node)  
  - Loop Over Items (SplitInBatches node)  

- **Node Details:**  
  - **Extract Notes/Emails**  
    - Type: Code (JavaScript)  
    - Role: Parses raw spreadsheet data to extract relevant notes and email text fields for AI input.  
    - Configuration: Custom JavaScript code filters or maps input data fields.  
    - Inputs: Output from "Get row(s) in sheet".  
    - Outputs: List of extracted textual data for batching.  
    - Failures: Parsing errors, unexpected data format, empty fields.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Splits extracted data into manageable batches (e.g., 5-10 items) to avoid API rate limits or payload size issues with Azure OpenAI.  
    - Configuration: Batch size parameter set (default or customized).  
    - Inputs: Array from "Extract Notes/Emails".  
    - Outputs: Sends batches to "Fetch Google Sheet Data" and "AI Agent".  
    - Failures: Batch size misconfiguration, empty batches.

---

#### 1.3 AI Processing

- **Overview:**  
  Processes the batched sales notes/emails using Azure OpenAI models wrapped in LangChain AI agents. Parses AI responses into structured feedback.

- **Nodes Involved:**  
  - Fetch Google Sheet Data (Google Sheets)  
  - AI Agent (LangChain agent)  
  - Azure OpenAI Chat Model (LM node)  
  - Edit Fields (Set node)  
  - Parse AI Response (Code node)  

- **Node Details:**  
  - **Fetch Google Sheet Data**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves additional or reference data from Google Sheets, possibly for context or enrichment for AI input.  
    - Configuration: Uses Google Sheets credentials.  
    - Inputs: Output from batch splitter "Loop Over Items".  
    - Outputs: Provides contextual data to "AI Agent1".  
    - Failures: Similar to prior Google Sheets node; includes connectivity and permission issues.

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Coordinates AI prompt construction and communication with Azure OpenAI chat models for analysis.  
    - Configuration: Connected downstream to "Azure OpenAI Chat Model".  
    - Inputs: Batch data and contextual info.  
    - Outputs: Raw AI response to "Edit Fields".  
    - Failures: API key/credential issues, rate limits, prompt errors.

  - **Azure OpenAI Chat Model**  
    - Type: Language Model Node (Azure OpenAI)  
    - Role: Executes the chat completion request to Azure OpenAI service.  
    - Configuration: Azure OpenAI credentials (endpoint, API key, deployment name), model selection (e.g., GPT-4 or GPT-3.5-turbo).  
    - Inputs: From "AI Agent".  
    - Outputs: AI-generated text response.  
    - Failures: Authentication errors, request timeouts, quota exceeded.

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Modifies or formats the AI response data for further parsing or storage.  
    - Configuration: Sets or clears fields, transforms data format.  
    - Inputs: AI Agent output.  
    - Outputs: To "Parse AI Response".  
    - Failures: Misconfiguration could lead to data loss.

  - **Parse AI Response**  
    - Type: Code (JavaScript)  
    - Role: Parses AI response text into structured JSON or spreadsheet-compatible format for saving.  
    - Configuration: Custom code extracting key insights and mapping them to columns.  
    - Inputs: Edited AI output.  
    - Outputs: Structured data for Google Sheets.  
    - Failures: Parsing errors if AI response format is unexpected.

---

#### 1.4 Data Storage

- **Overview:**  
  Saves the structured AI feedback into a Google Sheets document for record-keeping and further analysis.

- **Nodes Involved:**  
  - Save to Google Sheets (Google Sheets)  

- **Node Details:**  
  - **Save to Google Sheets**  
    - Type: Google Sheets (Write operation)  
    - Role: Appends or updates rows in a Google Sheet with AI-analyzed feedback data.  
    - Configuration: Spreadsheet ID, target sheet, and data mapping configured; uses Google credentials.  
    - Inputs: "Parse AI Response" output.  
    - Outputs: Loops back to "Loop Over Items" for processing next batch.  
    - Failures: Write permission errors, API limits, data format incompatibility.

---

#### 1.5 Reporting

- **Overview:**  
  Filters the latest AI-generated data and sends an email digest summarizing the sales analysis.

- **Nodes Involved:**  
  - Filter Recent Data (Code node)  
  - AI Agent1 (LangChain agent)  
  - Azure OpenAI Chat Model1 (LM node)  
  - Send Email Digest (Email Send node)  

- **Node Details:**  
  - **Filter Recent Data**  
    - Type: Code (JavaScript)  
    - Role: Filters the Google Sheets data to retrieve only recent entries (e.g., last day or week) for reporting.  
    - Configuration: Custom code with date filtering logic.  
    - Inputs: Output from "AI Agent1".  
    - Outputs: To "Send Email Digest".  
    - Failures: Date parsing errors, empty filtered results.

  - **AI Agent1**  
    - Type: LangChain AI Agent  
    - Role: Possibly refines or summarizes filtered data for inclusion in email.  
    - Configuration: Linked to "Azure OpenAI Chat Model1".  
    - Inputs: Data from "Fetch Google Sheet Data".  
    - Outputs: To "Filter Recent Data".  
    - Failures: Same AI-related errors as prior agent.

  - **Azure OpenAI Chat Model1**  
    - Type: Language Model Node (Azure OpenAI)  
    - Role: Runs chat completions for summarization or enhancement.  
    - Configuration: Similar to previous Azure OpenAI node but may have different prompt or parameters.  
    - Inputs: From "AI Agent1".  
    - Outputs: AI-generated summary.  
    - Failures: Same as above.

  - **Send Email Digest**  
    - Type: Email Send  
    - Role: Sends the summary report via email to configured recipients.  
    - Configuration: SMTP or other email credentials configured; email content likely includes AI summary.  
    - Inputs: Filtered and summarized data.  
    - Outputs: None (end of chain).  
    - Failures: SMTP auth errors, invalid email addresses, email sending failures.

---

### 3. Summary Table

| Node Name                | Node Type                     | Functional Role                                 | Input Node(s)               | Output Node(s)              | Sticky Note                              |
|--------------------------|-------------------------------|------------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Starts workflow manually                        | None                        | Get row(s) in sheet          |                                         |
| Get row(s) in sheet      | Google Sheets (Read)           | Retrieves raw sales data                        | When clicking ‘Execute workflow’ | Extract Notes/Emails         |                                         |
| Extract Notes/Emails     | Code                          | Extracts textual notes and emails               | Get row(s) in sheet          | Loop Over Items              |                                         |
| Loop Over Items          | SplitInBatches                | Batches data for AI processing                   | Extract Notes/Emails         | Fetch Google Sheet Data, AI Agent |                                         |
| Fetch Google Sheet Data  | Google Sheets (Read)           | Retrieves contextual data for AI                  | Loop Over Items              | AI Agent1                   |                                         |
| AI Agent                 | LangChain AI Agent            | Coordinates AI analysis with Azure OpenAI       | Loop Over Items              | Edit Fields                 |                                         |
| Azure OpenAI Chat Model  | Language Model (Azure OpenAI) | Performs AI chat completions                      | AI Agent                    | AI Agent                    |                                         |
| Edit Fields             | Set                           | Prepares AI response for parsing                 | AI Agent                    | Parse AI Response           |                                         |
| Parse AI Response        | Code                          | Parses AI text response into structured data    | Edit Fields                 | Save to Google Sheets       |                                         |
| Save to Google Sheets    | Google Sheets (Write)          | Saves AI feedback into spreadsheet               | Parse AI Response           | Loop Over Items             |                                         |
| AI Agent1                | LangChain AI Agent            | Summarizes or refines data for reporting         | Fetch Google Sheet Data     | Filter Recent Data          |                                         |
| Azure OpenAI Chat Model1 | Language Model (Azure OpenAI) | AI summary generation for email digest           | AI Agent1                   | AI Agent1                   |                                         |
| Filter Recent Data       | Code                          | Filters recent data entries for email digest     | AI Agent1                   | Send Email Digest           |                                         |
| Send Email Digest        | Email Send                    | Sends summarized report via email                 | Filter Recent Data          | None                       |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node named "When clicking ‘Execute workflow’".**  
   - No parameters needed. This node initiates the workflow manually.

2. **Add a Google Sheets node named "Get row(s) in sheet".**  
   - Set operation to "Read rows".  
   - Configure Google Sheets credentials (OAuth2).  
   - Specify the spreadsheet ID and the sheet name/range containing sales notes and emails.  
   - Connect its input from the Manual Trigger node.

3. **Add a Code node named "Extract Notes/Emails".**  
   - Write JavaScript to extract and map relevant note and email fields from the Google Sheets data.  
   - Input connected from "Get row(s) in sheet".  
   - Output produces an array of text items for AI processing.

4. **Add a SplitInBatches node named "Loop Over Items".**  
   - Configure batch size (e.g., 5 items).  
   - Connect input from "Extract Notes/Emails".  
   - Outputs to two nodes: "Fetch Google Sheet Data" and "AI Agent".

5. **Add a Google Sheets node named "Fetch Google Sheet Data".**  
   - Configure to read contextual or reference data needed for AI input.  
   - Use the same or appropriate Google Sheets credentials.  
   - Connect input from "Loop Over Items".

6. **Add a LangChain AI Agent node named "AI Agent".**  
   - Configure with appropriate LangChain agent settings.  
   - Connect input from "Loop Over Items" (batch data).  
   - Downstream connect its language model slot to an Azure OpenAI Chat Model node.

7. **Add an Azure OpenAI Chat Model node named "Azure OpenAI Chat Model".**  
   - Configure with Azure OpenAI credentials (endpoint, API key, deployment name).  
   - Select model (e.g., GPT-4 or GPT-3.5-turbo).  
   - Connect input from "AI Agent".  
   - Outputs to "Edit Fields".

8. **Add a Set node named "Edit Fields".**  
   - Configure to adjust or clean AI response fields as needed for parsing.  
   - Input from "AI Agent".  
   - Output to "Parse AI Response".

9. **Add a Code node named "Parse AI Response".**  
   - Create JavaScript code to parse AI response text into structured JSON or key-value pairs matching Google Sheets columns.  
   - Input from "Edit Fields".  
   - Output to "Save to Google Sheets".

10. **Add a Google Sheets node named "Save to Google Sheets".**  
    - Configure to append or update rows in a target spreadsheet with parsed AI data.  
    - Use appropriate credentials.  
    - Input from "Parse AI Response".  
    - Output loops back to "Loop Over Items" for processing remaining batches.

11. **Add a LangChain AI Agent node named "AI Agent1".**  
    - Configure similarly to "AI Agent".  
    - Input connects from "Fetch Google Sheet Data".  
    - Link its language model input to "Azure OpenAI Chat Model1".

12. **Add an Azure OpenAI Chat Model node named "Azure OpenAI Chat Model1".**  
    - Configure with Azure OpenAI credentials.  
    - Used for summarization or refinement.  
    - Connect input from "AI Agent1".  
    - Output to "Filter Recent Data".

13. **Add a Code node named "Filter Recent Data".**  
    - Write JavaScript to filter recent rows based on date or timestamp criteria.  
    - Input from "AI Agent1".  
    - Output to "Send Email Digest".

14. **Add an Email Send node named "Send Email Digest".**  
    - Configure SMTP or email service credentials.  
    - Set recipient addresses, subject, and email body (likely including AI-generated summary).  
    - Input from "Filter Recent Data".  
    - This node ends the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow leverages LangChain integration nodes for Azure OpenAI to facilitate complex AI prompt chaining.         | n8n documentation on LangChain nodes             |
| Ensure Google Sheets credentials have read/write permissions for the specified sheets to avoid auth errors.             | Google Sheets API documentation                   |
| Batch splitting is critical to avoid Azure OpenAI API rate limits and payload size constraints.                         | Azure OpenAI API best practices                   |
| Email node requires properly configured SMTP or external email service credentials (e.g., Gmail OAuth2 or SMTP server). | n8n Email node configuration guides               |

---

**Disclaimer:**  
The text provided is exclusively extracted from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.