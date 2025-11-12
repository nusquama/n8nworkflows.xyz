Track LLM Token Costs per Customer Using the Langchain Code Node 

https://n8nworkflows.xyz/workflows/track-llm-token-costs-per-customer-using-the-langchain-code-node--3440


# Track LLM Token Costs per Customer Using the Langchain Code Node 

### 1. Workflow Overview

This workflow implements a **token usage and cost tracking system for LLM (Large Language Model) calls per client/customer** using n8n's self-hosted Langchain Code node. It is designed for scenarios where multiple clients use an AI-powered service and it is necessary to monitor and bill their AI credit consumption accurately.

The workflow is structured into the following logical blocks:

- **1.1 Client Input Reception**: Handles PDF resume uploads from clients via a web form.
- **1.2 Data Parsing and Attribute Logging**: Extracts text from uploaded PDFs and adds metadata for tracking.
- **1.3 AI Data Extraction & Token Usage Tracking**: Uses a custom Langchain Code node subnode to run an LLM that extracts structured data from the resume while capturing token usage and costs.
- **1.4 Usage Data Logging to Google Sheets**: Stores the token usage and cost data per client in a Google Sheet.
- **1.5 Monthly Usage Aggregation and Billing**: Aggregates usage data monthly and optionally sends an invoice email to the client.

---

### 2. Block-by-Block Analysis

#### 2.1 Client Input Reception

**Overview:**  
This block accepts a PDF resume upload from the client using a form trigger. The form also collects a consent acknowledgement.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - *Type*: Form Trigger  
  - *Role*: Entry point for client uploads. Triggers workflow on form submit.  
  - *Configuration*:  
    - Form titled "CV Parsing Service" with fields: PDF file upload (required) and a multiselect dropdown for acknowledgement (required).  
    - Response mode set to "lastNode" to return final node output as form response.  
  - *Connections*: Outputs to "Parse PDF Upload".  
  - *Edge Cases*:  
    - File type must be PDF, upload failures or missing files will stop the workflow.  
    - Acknowledgement must be selected, else form submission is invalid.

---

#### 2.2 Data Parsing and Attribute Logging

**Overview:**  
Extracts text from the uploaded PDF and assigns additional metadata variables (workflow ID, execution ID, client ID) for usage logging.

**Nodes Involved:**  
- Parse PDF Upload  
- Logging Attributes

**Node Details:**  
- **Parse PDF Upload**  
  - *Type*: Extract From File  
  - *Role*: Extracts text content from the uploaded PDF file (binary data).  
  - *Configuration*: Operation set to "pdf", input binary property is the uploaded file.  
  - *Input*: From "On form submission" node.  
  - *Output*: JSON with extracted text for downstream use.  
  - *Edge Cases*: Corrupt or non-PDF files cause extraction failure.

- **Logging Attributes**  
  - *Type*: Set  
  - *Role*: Adds workflow-level metadata for logging purposes.  
  - *Configuration*: Sets fields:  
    - workflow_id = current workflow ID  
    - execution_id = current execution ID  
    - client_id = hardcoded "12345" (example client)  
  - *Input*: From "Parse PDF Upload".  
  - *Output*: Enriched data with metadata for AI processing.  
  - *Edge Cases*: Hardcoded client ID may need to be dynamic in real use; missing variables would impact logging.

---

#### 2.3 AI Data Extraction & Token Usage Tracking

**Overview:**  
Uses the Langchain Code node as a custom LLM subnode to process the extracted text, convert it into a structured JSON resume schema, and capture token usage and cost data via lifecycle hooks.

**Nodes Involved:**  
- Custom LLM Subnode  
- Extract Resume Data

**Node Details:**  
- **Custom LLM Subnode**  
  - *Type*: Langchain Code (custom LLM subnode)  
  - *Role*: Defines a custom OpenAI Chat LLM instance with lifecycle hooks to intercept token usage metadata on completion.  
  - *Configuration*:  
    - Uses OpenAI GPT-4o-mini model with hardcoded API key (replace with credential management).  
    - Token costs per million tokens configured: input tokens $0.150, output tokens $0.600.  
    - Callback `handleLLMEnd` extracts usage metadata from the LLM response, calculates costs, and sends a row object to the attached Google Sheets tool.  
  - *Input*: Receives Google Sheets tool as an input tool (via ai_tool connection).  
  - *Output*: Passes the LLM response to "Extract Resume Data".  
  - *Edge Cases*:  
    - API key must be valid; errors on authentication or rate limits must be handled externally.  
    - Failure to obtain usage metadata or Google Sheets append failure could cause data loss.  
  - *Version Requirements*: Requires self-hosted n8n (Langchain Code node is unavailable in cloud).  
  - *Sub-workflow*: Acts as a subnode within "Extract Resume Data" node.

- **Extract Resume Data**  
  - *Type*: Langchain Information Extractor  
  - *Role*: Uses LLM output to organize extracted text into a predefined JSON schema representing resume fields (name, email, education, skills, work, etc.).  
  - *Configuration*:  
    - Input text is the extracted PDF text from previous node.  
    - JSON schema is manually defined for resumes.  
  - *Input*: From "Logging Attributes" and LLM subnode output.  
  - *Output*: Provides structured JSON data for display and further processing.  
  - *Edge Cases*: LLM may fail to extract data correctly if input text is malformed or incomplete.

---

#### 2.4 Usage Data Logging to Google Sheets

**Overview:**  
Appends the token usage and cost data to a Google Sheets document to maintain a client usage log.

**Nodes Involved:**  
- Client Usage Log

**Node Details:**  
- **Client Usage Log**  
  - *Type*: Google Sheets Tool (append operation)  
  - *Role*: Receives usage metadata from the custom LLM subnode and appends it as a new row in the Google Sheet.  
  - *Configuration*:  
    - Document ID and Sheet name are set to a shared sheet ("Client Usage Log").  
    - Columns include date, workflow_id, execution_id, client_id, client_name, input_tokens, output_tokens, total_tokens, input_cost, output_cost, total_cost.  
    - Uses auto mapping of input data to columns.  
  - *Input*: Connected as a tool input to the "Custom LLM Subnode".  
  - *Edge Cases*:  
    - Google Sheets API credential must be valid.  
    - Network or API rate limits may cause failures.  
    - Sheet structure must align with mapping schema.

---

#### 2.5 Monthly Usage Aggregation and Billing

**Overview:**  
Periodically aggregates the monthly usage data from Google Sheets, calculates totals, and optionally sends an invoice email to the client.

**Nodes Involved:**  
- Every End of Month  
- Get Client Logs  
- Filter Last Month  
- Calculate Totals  
- Send Invoice

**Node Details:**  
- **Every End of Month**  
  - *Type*: Schedule Trigger  
  - *Role*: Triggers workflow monthly on the 31st day at 18:00.  
  - *Configuration*: Interval set to monthly trigger.  
  - *Output*: Triggers "Get Client Logs".  
  - *Edge Cases*: Months with fewer than 31 days may not trigger; consider adjusting trigger day.

- **Get Client Logs**  
  - *Type*: Google Sheets  
  - *Role*: Retrieves all usage log entries matching client_id "12345".  
  - *Configuration*: Filter rows where client_id equals "12345" from the same Google Sheet used previously.  
  - *Output*: Data passed to "Filter Last Month".  
  - *Edge Cases*: Filtering must work correctly; missing or malformed data may cause errors.

- **Filter Last Month**  
  - *Type*: Filter  
  - *Role*: Filters client usage rows for entries dated within the current month.  
  - *Configuration*: Dates checked between start and end of current month using expressions.  
  - *Output*: Passes filtered data to "Calculate Totals".  
  - *Edge Cases*: Date parsing failures or time zone mismatches.

- **Calculate Totals**  
  - *Type*: Summarize  
  - *Role*: Aggregates the total_cost and total_tokens fields from filtered data by summing.  
  - *Output*: Sends summary to "Send Invoice".  
  - *Edge Cases*: Empty data sets result in zero totals.

- **Send Invoice**  
  - *Type*: Gmail  
  - *Role*: Sends a billing email to the client with total token usage and cost summary including tax.  
  - *Configuration*:  
    - Recipient email hardcoded as "jim@example.com".  
    - Email subject and body use dynamic date and cost expressions.  
  - *Credential*: Gmail OAuth2 credentials required.  
  - *Edge Cases*: Email sending failures, invalid recipient address, or Gmail API issues.

---

#### 2.6 Displaying Extracted JSON to Client

**Overview:**  
Displays the extracted structured resume JSON back to the client in the form UI after processing.

**Nodes Involved:**  
- Display JSON Document

**Node Details:**  
- **Display JSON Document**  
  - *Type*: Form  
  - *Role*: Shows the JSON output of the resume extraction in a styled text box to the user.  
  - *Configuration*:  
    - Uses CSS to format the JSON output for readability.  
    - Completion title dynamically includes the uploaded filename.  
    - Completion message contains the formatted JSON stringified output.  
  - *Input*: Receives output from "Extract Resume Data".  
  - *Edge Cases*: Very large JSON may cause UI issues or timeouts.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                            | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                       |
|---------------------|--------------------------------|--------------------------------------------|-------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                   | Accept client PDF upload and acknowledgement |                         | Parse PDF Upload         | ## 1. Offer AI Service to Clients: Upload PDF and acknowledgement form.                         |
| Parse PDF Upload     | Extract From File              | Extract text from uploaded PDF              | On form submission       | Logging Attributes       |                                                                                                 |
| Logging Attributes   | Set                           | Assign workflow/execution/client metadata  | Parse PDF Upload         | Extract Resume Data      | ## 2. Gather External Variables to Send to Log                                                  |
| Custom LLM Subnode   | Langchain Code (custom LLM)   | Custom LLM with token usage tracking       | Client Usage Log (tool)  | Extract Resume Data      | ## 4. Use Custom LLM Subnode to Track Usage & Cost (includes Google Sheets tool attachment)     |
| Extract Resume Data  | Langchain Information Extractor | Convert extracted text into structured JSON | Logging Attributes, Custom LLM Subnode | Display JSON Document | ## 3. Provide AI Service: Organize resume data into JSON.                                       |
| Display JSON Document| Form                          | Show extracted JSON to client               | Extract Resume Data      |                         |                                                                                                 |
| Client Usage Log     | Google Sheets Tool            | Append token usage and cost data             | Custom LLM Subnode (tool)|                         | ### Update Workbook: tracks token usage and costs                                               |
| Every End of Month   | Schedule Trigger              | Monthly trigger for usage aggregation        |                         | Get Client Logs          | ## 5. Automatically Send Invoice at End of Month (Optional)                                     |
| Get Client Logs      | Google Sheets                 | Fetch usage logs for client                   | Every End of Month       | Filter Last Month        |                                                                                                 |
| Filter Last Month    | Filter                       | Filter usage logs to current month            | Get Client Logs          | Calculate Totals         |                                                                                                 |
| Calculate Totals     | Summarize                    | Aggregate total tokens and cost               | Filter Last Month        | Send Invoice             |                                                                                                 |
| Send Invoice         | Gmail                        | Send monthly invoice email                     | Calculate Totals         |                         |                                                                                                 |
| Sticky Note          | Sticky Note                  | Comments and explanations                      |                         |                         | Various sticky notes provide explanations and links.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure form: Title "CV Parsing Service"  
   - Add fields:  
     - File upload for PDFs (accept ".pdf", required)  
     - Multiselect dropdown for acknowledgement (required) with option "I acknowledge the use of this service will be added to my bill."  
   - Set response mode: "lastNode"

2. **Create "Parse PDF Upload" node**  
   - Type: Extract From File  
   - Operation: "pdf"  
   - Binary property: link to form upload field (e.g., "Upload_a_file")  
   - Connect input from "On form submission"

3. **Create "Logging Attributes" node**  
   - Type: Set  
   - Assign workflow_id = `{{$workflow.id}}`  
   - Assign execution_id = `{{$execution.id}}`  
   - Assign client_id = `"12345"` (replace with dynamic client ID if needed)  
   - Include all other incoming fields  
   - Connect input from "Parse PDF Upload"

4. **Create "Custom LLM Subnode" Langchain Code node**  
   - Type: Langchain Code (custom LLM subnode)  
   - Paste the provided JavaScript code that:  
     - Imports ChatOpenAI from Langchain  
     - Sets OpenAI API key and model ("gpt-4o-mini")  
     - Defines input/output token costs ($0.150/$0.600 per 1M tokens)  
     - Implements handleLLMEnd callback to extract usage metadata and append to Google Sheets via an attached tool  
   - Set input connection type as "ai_tool" (for Google Sheets tool)  
   - Set output connection type as "ai_languageModel"

5. **Create "Extract Resume Data" node**  
   - Type: Langchain Information Extractor  
   - Input text: `{{$json.text}}` from "Logging Attributes"  
   - Schema type: manual  
   - Paste the provided JSON schema describing resume structure (name, label, email, phone, location, work, education, skills, etc.)  
   - Attach the "Custom LLM Subnode" as an LLM subnode to this node

6. **Create "Display JSON Document" node**  
   - Type: Form  
   - Configure completion title: `Results for {{$('On form submission').item.json['Upload a file'][0].filename}}`  
   - Configure completion message: `{{ JSON.stringify($json.output, null, 2) }}`  
   - Add custom CSS for JSON formatting  
   - Connect input from "Extract Resume Data"

7. **Create "Client Usage Log" node**  
   - Type: Google Sheets Tool (append operation)  
   - Set Document ID to shared spreadsheet ID  
   - Set Sheet name to "Sheet1" (gid=0)  
   - Define columns matching usage metadata fields: date, workflow_id, execution_id, client_id, client_name, input_tokens, output_tokens, total_tokens, input_cost, output_cost, total_cost  
   - Use auto mapping of input data to columns  
   - Attach this node as a tool input to "Custom LLM Subnode" (ai_tool input)

8. **Create "Every End of Month" node**  
   - Type: Schedule Trigger  
   - Set interval to monthly trigger on day 31 at 18:00

9. **Create "Get Client Logs" node**  
   - Type: Google Sheets  
   - Use same Document ID and Sheet name as "Client Usage Log"  
   - Set filter to retrieve rows where client_id = "12345" (or dynamic client ID)  
   - Connect input from "Every End of Month"

10. **Create "Filter Last Month" node**  
    - Type: Filter  
    - Conditions:  
      - Date >= start of current month (`{{$now.startOf('month')}}`)  
      - Date <= end of current month (`{{$now.endOf('month')}}`)  
    - Connect input from "Get Client Logs"

11. **Create "Calculate Totals" node**  
    - Type: Summarize  
    - Fields to summarize: sum of `total_cost` and sum of `total_tokens`  
    - Connect input from "Filter Last Month"

12. **Create "Send Invoice" node**  
    - Type: Gmail  
    - Configure recipient email (e.g., "jim@example.com")  
    - Email subject: "Invoice for {{ $now.monthLong }} {{ $now.year }}"  
    - Email body: includes total tokens, cost, tax (20%), and total payable with dynamic expressions  
    - Use Gmail OAuth2 credentials  
    - Connect input from "Calculate Totals"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                                                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| This template is for **self-hosted n8n only**; the Langchain Code node is not available in the cloud version.                                                                 | Sticky Note: "SELF-HOSTED N8N ONLY"                                                                                                                      |
| Learn more about the Langchain Code node and how to customize it here: [Langchain Code Node Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.code/) | Sticky Note4 content                                                                                                                                       |
| Example client usage log spreadsheet: [Google Sheet Link](https://docs.google.com/spreadsheets/d/1AR5mrxz2S6PjAKVM0edNG-YVEc6zKL7aUxHxVcffnlw/edit?usp=sharing)                   | Provided in workflow description and sticky notes                                                                                                        |
| Join the n8n community for help: [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)                                                           | Sticky Note6 content                                                                                                                                       |
| The invoice email logic includes a fixed 20% tax calculation and a 14-day payment term; adapt as per your billing policies.                                                   | Observed in "Send Invoice" node configuration                                                                                                            |
| The client_id is hardcoded in multiple nodes as "12345"; replace or parameterize this for multi-client production use.                                                      | Noted in "Logging Attributes" and "Get Client Logs" nodes                                                                                                |

---

This document fully captures the workflow structure, node configurations, logic flow, and key considerations for replication, modification, and error anticipation. It also provides all resource links preserved from sticky notes.