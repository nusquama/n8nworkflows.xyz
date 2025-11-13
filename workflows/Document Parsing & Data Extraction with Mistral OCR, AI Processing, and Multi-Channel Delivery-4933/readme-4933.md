Document Parsing & Data Extraction with Mistral OCR, AI Processing, and Multi-Channel Delivery

https://n8nworkflows.xyz/workflows/document-parsing---data-extraction-with-mistral-ocr--ai-processing--and-multi-channel-delivery-4933


# Document Parsing & Data Extraction with Mistral OCR, AI Processing, and Multi-Channel Delivery

### 1. Workflow Overview

This workflow is designed for automated document parsing and data extraction from PDFs using Mistral OCR services, followed by AI-based processing and multi-channel delivery of extracted information. The main goal is to receive documents (e.g., invoices), parse and extract structured data, refine the data using AI agents, and distribute the processed content via Telegram and Gmail.

Logical blocks of the workflow:

- **1.1 Input Reception & File Download**: Receives documents via Telegram Trigger or form submission and downloads the associated PDF file.

- **1.2 OCR Processing with Mistral**: Sends the downloaded PDF to Mistral OCR API, obtains a signed URL, and retrieves parsed invoice data.

- **1.3 AI Data Processing**: Uses Langchain AI Agents with Mistral and OpenAI chat models to interpret and refine the extracted data.

- **1.4 Data Formatting and Editing**: Applies field edits and Markdown formatting to prepare the data for delivery.

- **1.5 Multi-Channel Delivery**: Sends the final processed information via Telegram messages and Gmail emails, including confirmation messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Download

- **Overview:** This block captures incoming documents via Telegram or form submissions, then downloads the PDF file for further processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Telegram (two separate Telegram nodes for sending messages)  
  - On form submission  
  - Download File  
  - Code  

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram messages  
    - Configuration: Listens for incoming Telegram messages containing documents or commands  
    - Input: External Telegram webhook  
    - Output: Passes data to Telegram node  
    - Edge cases: Invalid message format, unsupported file types, network issues  
    - No sub-workflows  

  - **Telegram**  
    - Type: Telegram message sender  
    - Configuration: Sends messages back to users, e.g., confirmations or error notifications  
    - Input: Triggered by Telegram Trigger node or other workflow parts  
    - Output: None  
    - Edge cases: Telegram API errors, rate limits  

  - **On form submission**  
    - Type: Webhook trigger for form inputs  
    - Configuration: Triggers when a form is submitted (possibly another input method)  
    - Input: External webhook  
    - Output: No direct continuation shown here but logically it can feed into the workflow  
    - Edge cases: Missing fields, invalid form data  

  - **Download File**  
    - Type: HTTP Request  
    - Configuration: Downloads the PDF file from a URL (likely from Telegram message or form)  
    - Input: Receives URL from the trigger nodes or code node  
    - Output: Passes file binary data to "Code" node  
    - Edge cases: Download failure, URL invalid, timeout  

  - **Code**  
    - Type: Custom JavaScript/Node.js code execution  
    - Configuration: Processes or prepares data before sending to Mistral OCR  
    - Input: Binary file from "Download File"  
    - Output: Passes processed data to "send pdf to Mistral" node  
    - Edge cases: Script errors, unexpected data format  

---

#### 2.2 OCR Processing with Mistral

- **Overview:** Sends the PDF to Mistral OCR API, retrieves a signed URL for the processed document, then fetches parsed invoice data.

- **Nodes Involved:**  
  - send pdf to Mistral (HTTP Request)  
  - mistral - get signed url (HTTP Request)  
  - Get Parsed Invoice (HTTP Request)  

- **Node Details:**

  - **send pdf to Mistral**  
    - Type: HTTP Request  
    - Configuration: Sends the PDF file to Mistral’s OCR API endpoint for processing  
    - Input: Receives PDF binary from "Code" node  
    - Output: Provides response (likely job ID or token) to "mistral - get signed url"  
    - Edge cases: API authentication errors, file too large, network failures  

  - **mistral - get signed url**  
    - Type: HTTP Request  
    - Configuration: Requests a signed URL from Mistral to access the processed document or results  
    - Input: Receives job ID/token from "send pdf to Mistral"  
    - Output: Signed URL passed to "Get Parsed Invoice"  
    - Edge cases: Expired tokens, invalid job IDs  

  - **Get Parsed Invoice**  
    - Type: HTTP Request  
    - Configuration: Downloads or retrieves structured invoice data from Mistral using the signed URL  
    - Input: Signed URL from "mistral - get signed url"  
    - Output: Passes invoice data to AI Agent for interpretation  
    - Edge cases: Malformed data, delayed processing  

---

#### 2.3 AI Data Processing

- **Overview:** Uses AI agents powered by Langchain with Mistral and OpenAI chat models to interpret and improve the parsed invoice data, enabling sophisticated data understanding and extraction.

- **Nodes Involved:**  
  - Mistral Cloud Chat Model1 (AI Language Model)  
  - AI Agent  
  - AI Agent1  
  - OpenAI Chat Model  

- **Node Details:**

  - **Mistral Cloud Chat Model1**  
    - Type: Langchain AI Language Model (Mistral)  
    - Configuration: Processes raw parsed data to extract or infer additional details  
    - Input: Connected as AI language model reference to "AI Agent"  
    - Output: Provides AI-enhanced data to "AI Agent"  
    - Edge cases: API limits, model errors, incorrect prompt formatting  

  - **AI Agent**  
    - Type: Langchain AI Agent  
    - Configuration: Uses Mistral Cloud Chat Model1 as underlying model  
    - Input: Parsed invoice data from "Get Parsed Invoice"  
    - Output: Outputs refined data to "Edit Fields"  
    - Edge cases: Agent execution errors, unexpected data shapes  

  - **OpenAI Chat Model**  
    - Type: Langchain AI Language Model (OpenAI)  
    - Configuration: Provides alternative or complementary AI processing capabilities  
    - Input: Feeds into "AI Agent1" as language model  
    - Output: AI responses passed to "AI Agent1"  
    - Edge cases: API key invalid, rate limits, network failures  

  - **AI Agent1**  
    - Type: Langchain AI Agent  
    - Configuration: Uses OpenAI Chat Model to further process or finalize content  
    - Input: Data from "Edit Fields2" and OpenAI Chat Model  
    - Output: Passes final content to delivery formatting nodes  
    - Edge cases: AI processing failures, prompt issues  

---

#### 2.4 Data Formatting and Editing

- **Overview:** Sets or edits specific data fields and transforms the AI output into Markdown-formatted content, preparing it for sending to users.

- **Nodes Involved:**  
  - Edit Fields  
  - Markdown  
  - Edit Fields2  

- **Node Details:**

  - **Edit Fields**  
    - Type: Set node  
    - Configuration: Modifies or adds fields to the AI Agent output, likely cleaning or structuring data  
    - Input: Output from "AI Agent"  
    - Output: Passes edited data to "Markdown"  
    - Edge cases: Expression errors, missing expected fields  

  - **Markdown**  
    - Type: Markdown node  
    - Configuration: Converts structured data into Markdown format to enhance readability in messages or emails  
    - Input: From "Edit Fields"  
    - Output: Passes Markdown content to "Edit Fields2"  
    - Edge cases: Malformed Markdown, unsupported characters  

  - **Edit Fields2**  
    - Type: Set node  
    - Configuration: Final adjustments to the content before AI Agent1 processing, possibly adds metadata or formats  
    - Input: Markdown output  
    - Output: Passes to "AI Agent1"  
    - Edge cases: Expression failures  

---

#### 2.5 Multi-Channel Delivery

- **Overview:** Delivers the finalized processed data via Telegram and Gmail to end users, including sending confirmation messages.

- **Nodes Involved:**  
  - Telegram1  
  - Gmail  
  - Send Confirmation  

- **Node Details:**

  - **Telegram1**  
    - Type: Telegram message sender  
    - Configuration: Sends the final message with extracted and formatted data to Telegram users  
    - Input: From "AI Agent1"  
    - Output: None  
    - Edge cases: Telegram API errors, rate limiting  

  - **Gmail**  
    - Type: Gmail node (email sender)  
    - Configuration: Sends an email containing the processed document data or invoice details  
    - Input: Receives data from "AI Agent1"  
    - Output: None  
    - Credential: Requires Gmail OAuth2 credentials  
    - Edge cases: Authentication errors, email sending failures  

  - **Send Confirmation**  
    - Type: Telegram message sender  
    - Configuration: Sends confirmation messages to users after processing completion  
    - Input: Triggered after sending main messages  
    - Output: None  
    - Edge cases: Telegram API errors  

---

### 3. Summary Table

| Node Name              | Node Type                            | Functional Role                       | Input Node(s)               | Output Node(s)             | Sticky Note                              |
|------------------------|------------------------------------|------------------------------------|-----------------------------|----------------------------|-----------------------------------------|
| Telegram Trigger       | Telegram Trigger                   | Input reception from Telegram       | External webhook             | Telegram                   |                                         |
| Telegram               | Telegram                          | Send Telegram messages              | Telegram Trigger             | None                       |                                         |
| On form submission     | Form Trigger                     | Input reception from form           | External webhook             | None                       |                                         |
| Download File          | HTTP Request                     | Download PDF file                   | External triggers/Code       | Code                       |                                         |
| Code                   | Code                             | Prepare file/data for OCR           | Download File                | send pdf to Mistral        |                                         |
| send pdf to Mistral    | HTTP Request                     | Send PDF to Mistral OCR             | Code                        | mistral - get signed url   |                                         |
| mistral - get signed url| HTTP Request                     | Get signed URL for processed data   | send pdf to Mistral          | Get Parsed Invoice         |                                         |
| Get Parsed Invoice     | HTTP Request                     | Retrieve parsed invoice data        | mistral - get signed url     | AI Agent                   |                                         |
| Mistral Cloud Chat Model1 | Langchain LM Chat Model          | AI language model (Mistral)         |                             | AI Agent (languageModel)   |                                         |
| AI Agent               | Langchain AI Agent               | Process parsed data with Mistral AI | Get Parsed Invoice           | Edit Fields                |                                         |
| Edit Fields            | Set                              | Edit/format AI output fields        | AI Agent                    | Markdown                   |                                         |
| Markdown               | Markdown                         | Format content to Markdown           | Edit Fields                 | Edit Fields2               |                                         |
| Edit Fields2           | Set                              | Final content edits before AI       | Markdown                    | AI Agent1                  |                                         |
| OpenAI Chat Model      | Langchain LM Chat Model          | AI language model (OpenAI)           |                             | AI Agent1 (languageModel)  |                                         |
| AI Agent1              | Langchain AI Agent               | Final AI processing with OpenAI     | Edit Fields2, OpenAI Chat Model| Telegram1, Gmail          |                                         |
| Telegram1              | Telegram                        | Send final Telegram message         | AI Agent1                   | None                       |                                         |
| Gmail                  | Gmail                           | Send email with processed data      | AI Agent1                   | None                       | Requires Gmail OAuth2 credentials       |
| Send Confirmation      | Telegram                        | Send confirmation message           |                             | None                       |                                         |
| Telegram               | Telegram                        | Secondary Telegram sender            | Telegram Trigger            | None                       |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Triggers:**  
   - Add a **Telegram Trigger** node configured to listen for incoming messages containing files/documents. Configure with your Telegram bot token and webhook.  
   - Add a **Form Trigger** node for form submissions if applicable, configure webhook URL.

2. **Download File Node:**  
   - Add an **HTTP Request** node named "Download File". Configure it to download the PDF file from a URL extracted from the Telegram message or form submission data using expressions.

3. **Code Node:**  
   - Add a **Code** node to process or prepare the downloaded PDF binary data for sending to Mistral OCR. Configure the code to handle input binary data and format it as required by Mistral’s API.

4. **Mistral OCR Nodes:**  
   - Add an **HTTP Request** node "send pdf to Mistral" to upload the PDF to Mistral OCR API. Configure with method POST, API endpoint URL, and include necessary authentication headers and binary payload.  
   - Add an **HTTP Request** node "mistral - get signed url" to request a signed URL to access the OCR results, using the response from the previous node.  
   - Add an **HTTP Request** node "Get Parsed Invoice" to retrieve the structured invoice data using the signed URL.

5. **AI Processing Nodes:**  
   - Add a **Langchain LM Chat Model** node configured for Mistral Cloud Chat Model ("Mistral Cloud Chat Model1"). Provide necessary API keys or credentials.  
   - Add a **Langchain AI Agent** node "AI Agent", connected to "Get Parsed Invoice" and linked to the Mistral AI model node. Configure prompts or chains as needed for invoice data interpretation.  
   - Add a **Set** node "Edit Fields" to clean or format AI output fields.  
   - Add a **Markdown** node to convert structured data into Markdown format.  
   - Add a **Set** node "Edit Fields2" for final field adjustments before passing to the next AI agent.  
   - Add a **Langchain LM Chat Model** node for OpenAI Chat Model; provide OpenAI API credentials.  
   - Add a **Langchain AI Agent** node "AI Agent1" connected to "Edit Fields2" and OpenAI Chat Model node for final data processing.

6. **Delivery Nodes:**  
   - Add a **Telegram** node "Telegram1" to send the final message back to the user’s Telegram account. Configure with bot token and chat ID expressions.  
   - Add a **Gmail** node to send an email with the processed invoice data. Configure with Gmail OAuth2 credentials, set sender and recipient emails, subject, and message body using workflow data.  
   - Add a **Telegram** node "Send Confirmation" to send confirmation messages after processing completes.

7. **Connect Nodes in the Correct Order:**  
   - Telegram Trigger → Download File → Code → send pdf to Mistral → mistral - get signed url → Get Parsed Invoice → AI Agent → Edit Fields → Markdown → Edit Fields2 → AI Agent1 → Telegram1 & Gmail → Send Confirmation.

8. **Credential Setup:**  
   - Configure Telegram credentials with Bot Token and Webhook.  
   - Configure Mistral OCR API credentials (API Key or token).  
   - Configure OpenAI credentials for Chat Model nodes.  
   - Configure Gmail OAuth2 credentials for email sending.

9. **Default Values and Constraints:**  
   - Set reasonable timeouts and retries on HTTP nodes to manage network issues.  
   - Validate input files to ensure PDF format before processing.  
   - Implement error handling or fallback messages for AI failures or API errors.

10. **Optional:**  
    - Add sticky notes or documentation nodes for clarity.  
    - Enable workflow execution order as needed (default "v1").

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow uses both Mistral OCR and Langchain AI agents to combine OCR and AI-based semantic processing.  | Workflow description                                 |
| Gmail node requires OAuth2 credentials with appropriate scopes for sending emails on behalf of the user.     | Gmail node credential requirement                     |
| Telegram nodes require bot tokens and webhook URLs properly configured in Telegram Bot API dashboard.         | Telegram API setup                                   |
| The workflow demonstrates multi-channel delivery of processed data: Telegram messages and email notifications. | Functional design                                    |
| For Langchain AI Agents, ensure API keys and environment variables are securely configured in n8n credentials. | Security best practices                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.