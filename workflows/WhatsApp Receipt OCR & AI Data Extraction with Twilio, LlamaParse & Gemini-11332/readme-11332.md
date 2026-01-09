WhatsApp Receipt OCR & AI Data Extraction with Twilio, LlamaParse & Gemini

https://n8nworkflows.xyz/workflows/whatsapp-receipt-ocr---ai-data-extraction-with-twilio--llamaparse---gemini-11332


# WhatsApp Receipt OCR & AI Data Extraction with Twilio, LlamaParse & Gemini

### 1. Workflow Overview

This workflow automates the extraction of structured data from receipts or invoices sent via WhatsApp. It is designed for use cases where users submit images or PDFs of Australian receipts through WhatsApp, and the system processes the documents using OCR and AI to extract key fields such as vendor, cost, tax, ABN, date, and currency. Extracted data is stored in Google Sheets and receipt files are uploaded to Google Drive. Users receive WhatsApp messages with extracted information and a prompt to confirm data accuracy.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Routing:** Receives WhatsApp messages with media attachments and routes based on user replies.
- **1.2 Receipt OCR Upload & Polling:** Uploads receipt files to LlamaParse OCR and polls for processing results.
- **1.3 AI-Based Data Extraction:** Uses AI models Gemini and OpenRouter Claude to extract structured receipt data.
- **1.4 Data Normalization & Tax Calculation:** Normalizes extracted fields, especially date formatting; calculates tax if not explicitly provided.
- **1.5 Data Storage:** Stores extracted data and metadata in Google Sheets and uploads receipt files to Google Drive.
- **1.6 WhatsApp User Interaction:** Sends WhatsApp messages to users with data summaries and confirmation prompts; handles user quick replies.
- **1.7 Error Handling & Retry:** Manages polling delays and retries for OCR completion and Twilio API calls.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Routing

- **Overview:**  
  This block receives incoming WhatsApp messages via webhook and routes them according to the user’s quick reply or message payload.

- **Nodes Involved:**  
  - Webhook3  
  - Switch1  
  - HTTP Request20 (Yes confirmation message)  
  - HTTP Request21 (No confirmation message)  
  - HTTP Request7 (Default fallback message)

- **Node Details:**  
  - **Webhook3** (Webhook)  
    - Receives POST requests from Twilio WhatsApp webhook with message data including media URLs and user replies.  
    - Trigger for entire workflow.  
    - Input: HTTP POST from Twilio webhook.  
    - Output: JSON containing message body, media URLs, message SIDs, and quick reply payloads.  
    - Failures: Network issues, malformed webhook requests.

  - **Switch1** (Switch)  
    - Routes flow based on `ButtonPayload` value extracted from webhook JSON.  
    - Routes:  
      - "1" → HTTP Request20 (positive confirmation)  
      - "2" → HTTP Request21 (negative confirmation)  
      - "none" → HTTP Request7 (default message requesting re-upload)  
    - Edge cases: Unexpected or missing ButtonPayload values.

  - **HTTP Request20 / HTTP Request21 / HTTP Request7** (Twilio API POST)  
    - Send WhatsApp messages back to user.  
    - Use Twilio preconfigured OAuth2 credentials.  
    - Body contains predefined texts for confirmation or re-upload prompts.  
    - Failures: Twilio API errors, authentication issues, rate limits.

---

#### 1.2 Receipt OCR Upload & Polling

- **Overview:**  
  Uploads the media file received from WhatsApp to LlamaParse OCR API for text extraction and polls the job until completion.

- **Nodes Involved:**  
  - Append row in sheet2  
  - HTTP Request5 (Download media from Twilio)  
  - Upload file2 (Google Drive upload)  
  - HTTP Request29 (LlamaParse file upload)  
  - HTTP Request28 (Job status polling)  
  - Wait (Delays between polls)  
  - HTTP Request30 (Fetch OCR result in markdown)  
  - If5 (Check job success)

- **Node Details:**  
  - **Append row in sheet2** (Google Sheets Append)  
    - Logs the incoming message SID to Google Sheets as an initial record.  
    - Credentials: Google Sheets OAuth2.  
    - Potential failures: Sheet access denied, quota exceeded.

  - **HTTP Request5** (Download media)  
    - Downloads the file from Twilio MediaUrl0 URL with Twilio OAuth credentials.  
    - Timeout set to 180s; retries enabled with 2s wait.  
    - Failure modes: Network timeout, invalid URL, auth error.

  - **Upload file2** (Google Drive)  
    - Uploads the downloaded file to Google Drive under “My Drive.”  
    - File is named using the WhatsApp MessageSid plus original filename.  
    - Credentials: Google Drive OAuth2.  
    - Failures: Storage limits, permission errors.

  - **HTTP Request29** (LlamaParse OCR file upload)  
    - Uploads the file binary to LlamaParse API with multipart form-data and query parameters specifying parse mode, model, and invoice preset.  
    - Uses HTTP Bearer token (credential: Placid).  
    - Failure types: Authentication failure, API downtime, invalid file format.

  - **HTTP Request28** (LlamaParse job status polling)  
    - Polls the job status endpoint using the job ID from the upload response.  
    - Authentication same as upload.  
    - Output checked by If5 node.  
    - Failure: Job not found, timeout, API errors.

  - **Wait**  
    - Waits 2 seconds between status polls to avoid spamming API.  
    - Prevents rate limit breaches.

  - **HTTP Request30** (Fetch OCR result in markdown)  
    - Retrieves the final OCR text in markdown format from LlamaParse after job completion.  
    - Used as input for AI extraction.  
    - Failure: Incomplete job, timeout.

  - **If5** (Check job status)  
    - Checks if LlamaParse job status is "SUCCESS".  
    - If yes, proceeds to fetch result; if no, continues waiting and polling.  
    - Failure: Unexpected job status, infinite wait loop if job fails.

---

#### 1.3 AI-Based Data Extraction

- **Overview:**  
  Processes the OCR text with a LangChain LLM chain using Google Gemini and OpenRouter Claude AI models to extract structured receipt data (vendor, cost, tax, date, currency, ABN).

- **Nodes Involved:**  
  - Basic LLM Chain3  
  - Google Gemini Chat Model5  
  - OpenRouter Chat Model  
  - Structured Output Parser3  
  - Google Gemini Chat Model6

- **Node Details:**  
  - **Basic LLM Chain3** (LangChain LLM Chain)  
    - Uses a prompt specifically designed to extract structured data from Australian receipts, with detailed rules for tax and vendor identification.  
    - Supports fallback LLM (OpenRouter Claude) if Gemini API is unavailable.  
    - Input: markdown OCR text.  
    - Output: JSON with extracted fields.  
    - Failure: API quota, prompt errors, inconsistent output.

  - **Google Gemini Chat Model5 and 6** (AI LLM Models)  
    - Gemini 2.5-pro model used to parse and extract data.  
    - Connected as language model credentials via Google Palm API.  
    - Model 6 feeds into the Structured Output Parser3 node.  
    - Failures: API latency, auth failures.

  - **OpenRouter Chat Model**  
    - Secondary AI model (Anthropic Claude Sonnet 4.5) used as fallback for Basic LLM Chain3.  
    - Auth via OpenRouter API key.  
    - Failure mode: fallback not triggered if primary succeeds.

  - **Structured Output Parser3** (Output Parser)  
    - Parses AI output to JSON with schema validation, auto-fixing minor issues.  
    - Ensures fields conform to expected types and formatting.  
    - Failure: Parsing errors if AI output deviates significantly.

---

#### 1.4 Data Normalization & Tax Calculation

- **Overview:**  
  Normalizes extracted data, notably date formatting, and calculates tax if missing or ambiguous.

- **Nodes Involved:**  
  - Code4 (Date normalization)  
  - If (Tax check)  
  - Code5 (Tax calculation if needed)  
  - Merge2 (Merge data)

- **Node Details:**  
  - **Code4** (JavaScript)  
    - Normalizes date strings to DD/MM/YYYY format using robust parsing logic.  
    - Extracts fields safely from AI output, handles multiple formats and invalid dates.  
    - Edge cases: invalid date strings, unexpected formats.

  - **If** (Tax field validation)  
    - Checks if Tax field string contains "cost" or "found" indicating uncertainty or absence.  
    - Routes to tax calculation if needed.  
    - Failure: Incorrect conditional logic may skip tax calculation.

  - **Code5** (Tax calculation)  
    - Calculates tax as cost/11 (standard Australian GST) if tax is not explicitly provided.  
    - Updates Tax field with computed value.  
    - Edge cases: cost not numeric or zero.

  - **Merge2** (Merge)  
    - Combines outputs from tax calculation and normalized data for further processing.

---

#### 1.5 Data Storage

- **Overview:**  
  Stores extracted and normalized data into Google Sheets and uploads receipt files to Google Drive, sharing the file publicly.

- **Nodes Involved:**  
  - Append row in sheet2 (initial log)  
  - Upload file2 (Google Drive upload)  
  - Share file1 (Google Drive permission update)  
  - Update row in sheet4, sheet5, sheet9, sheet10, sheet11 (Google Sheets updates)  
  - Merge3 (Merge data before final updates)

- **Node Details:**  
  - **Append row in sheet2**  
    - Logs incoming message SID to sheet for tracking.  
    - Failure: Sheet permissions or quota issues.

  - **Upload file2**  
    - Uploads the receipt image or PDF to Google Drive using OAuth2 credentials.  
    - Filename combines WhatsApp message SID and original filename.  
    - Failures: Storage limits, API errors.

  - **Share file1**  
    - Sets file sharing permission to "reader" for "anyone" to enable public access via link.  
    - Failure: Permission denied, API limits.

  - **Update row in sheet4, sheet5, sheet9, sheet10, sheet11**  
    - Update specific fields in Google Sheets rows matching message SIDs or quick reply IDs.  
    - sheet4 updates with link to Google Drive file.  
    - sheet5 updates with extracted vendor, cost, tax, date, ABN, and status "Not confirmed yet."  
    - sheet9 and sheet10 update rows based on user quick reply (Yes/No confirmation).  
    - sheet11 records phone number and quick reply ID.  
    - Failure: Mismatched keys, concurrency issues.

  - **Merge3**  
    - Merges updated rows for final processing and synchronization.

---

#### 1.6 WhatsApp User Interaction

- **Overview:**  
  Sends WhatsApp messages to users with extracted receipt details and asks for confirmation whether the data is accurate using quick replies.

- **Nodes Involved:**  
  - HTTP Request6 (Send extracted data summary)  
  - HTTP Request19 (Follow-up message)  
  - HTTP Request20 / HTTP Request21 (Confirmation replies)  
  - HTTP Request7 (Fallback message)  
  - HTTP Request22, 24, 28 (Typing indicators)

- **Node Details:**  
  - **HTTP Request6**  
    - Sends a WhatsApp message to user with extracted receipt details (vendor, cost, tax, date, ABN).  
    - Uses Twilio API credentials.  
    - Failure: Twilio API errors, message formatting issues.

  - **HTTP Request19**  
    - Sends an additional message with templated content.  
    - Used for interaction continuity.

  - **HTTP Request20 / HTTP Request21**  
    - Sends "Thank you for confirming!" or "Please re-upload the receipt file." messages based on user reply.  
    - Uses Twilio WhatsApp messaging templates with quick reply buttons.  
    - Failure: Template not approved by WhatsApp, API errors.

  - **HTTP Request7, 22, 24, 28**  
    - Send "typing" indicator to user to improve UX during processing delays.  
    - Retried on failure for robustness.

---

#### 1.7 Error Handling & Retry

- **Overview:**  
  Implements retry and wait mechanisms for handling asynchronous operations and transient failures.

- **Nodes Involved:**  
  - Wait (delays between OCR polling)  
  - Retry settings on HTTP Request nodes (Twilio and LlamaParse calls)  
  - If5 (conditional looping for OCR job success)

- **Node Details:**  
  - **Wait**  
    - Pauses 2 seconds before polling LlamaParse job status again.  
    - Prevents excessive API calls and rate limits.

  - **HTTP Request nodes with retryOnFail**  
    - Twilio and LlamaParse API calls have retry enabled with delay between tries.  
    - Failure modes handled: network timeouts, temporary auth issues, rate limits.

  - **If5**  
    - Controls looping for polling OCR job status until success or error.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                                    | Input Node(s)            | Output Node(s)              | Sticky Note                                        |
|-------------------------|---------------------------------|---------------------------------------------------|--------------------------|-----------------------------|---------------------------------------------------|
| Webhook3                | Webhook                         | Receive WhatsApp messages & media                  | -                        | Switch1                     |                                                   |
| Switch1                 | Switch                         | Route based on user quick reply                     | Webhook3                 | HTTP Request20, 21, 7       |                                                   |
| HTTP Request20          | HTTP Request (Twilio)           | Send confirmation "Thank you" message              | Switch1                  | Update row in sheet9        |                                                   |
| HTTP Request21          | HTTP Request (Twilio)           | Send "Please re-upload" message                     | Switch1                  | Update row in sheet10       |                                                   |
| HTTP Request7           | HTTP Request (Twilio)           | Send fallback message                               | Switch1                  | Append row in sheet2        |                                                   |
| Append row in sheet2    | Google Sheets Append            | Log incoming message SID                            | HTTP Request7             | HTTP Request5               |                                                   |
| HTTP Request5           | HTTP Request (Twilio)           | Download receipt file from Twilio                   | Append row in sheet2      | Upload file2, HTTP Request29 |                                                   |
| Upload file2            | Google Drive Upload             | Upload receipt file to Google Drive                 | HTTP Request5             | Share file1                 |                                                   |
| Share file1             | Google Drive Share              | Make uploaded file publicly accessible              | Upload file2              | Update row in sheet4        |                                                   |
| Update row in sheet4    | Google Sheets Update            | Update row with Google Drive file link              | Share file1               | Merge3                     |                                                   |
| HTTP Request29          | HTTP Request (LlamaParse)       | Upload file to LlamaParse OCR API                    | HTTP Request5             | HTTP Request28, HTTP Request17 | Sticky Note1: OCR pooling mechanism using LlamaParse |
| HTTP Request28          | HTTP Request (LlamaParse)       | Poll OCR job status                                  | HTTP Request29            | If5                        |                                                   |
| Wait                    | Wait                           | Wait 2 seconds between OCR job polls                 | If5 (fail branch)         | HTTP Request28             |                                                   |
| If5                     | If                             | Check if OCR job status is SUCCESS                   | HTTP Request28            | HTTP Request30, Wait       |                                                   |
| HTTP Request30          | HTTP Request (LlamaParse)       | Fetch OCR result in markdown                          | If5 (success branch)      | Basic LLM Chain3           |                                                   |
| Basic LLM Chain3        | LangChain LLM Chain             | Extract structured receipt data from OCR markdown   | HTTP Request30            | Code4, HTTP Request18       | Sticky Note3: extracted values, no yes/no buttons |
| HTTP Request18          | HTTP Request (Twilio)           | Send typing indicator                               | Basic LLM Chain3          | -                         |                                                   |
| Code4                   | Code                           | Normalize date and extract fields                    | Basic LLM Chain3          | If                        | Sticky Note4: Calculate tax if not specified on the receipt |
| If                      | If                             | Check tax field validity                             | Code4                    | Code5, Merge2              |                                                   |
| Code5                   | Code                           | Calculate tax as cost/11 if missing                   | If (tax invalid branch)   | Merge2                     |                                                   |
| Merge2                  | Merge                          | Combine normalized and tax-calculated data           | Code5, If                 | Update row in sheet5       |                                                   |
| Update row in sheet5    | Google Sheets Update            | Update Google Sheet row with extracted data          | Merge2                    | HTTP Request6              |                                                   |
| HTTP Request6           | HTTP Request (Twilio)           | Send WhatsApp message with extracted receipt data    | Update row in sheet5      | HTTP Request19             |                                                   |
| HTTP Request19          | HTTP Request (Twilio)           | Follow-up WhatsApp message                            | HTTP Request6             | Update row in sheet11      |                                                   |
| Update row in sheet11   | Google Sheets Update            | Update row with phone and quick reply info           | HTTP Request19            | Merge3                     |                                                   |
| Merge3                  | Merge                          | Final data merge for sheet updates                    | Update row in sheet4,11   | -                         |                                                   |
| HTTP Request22          | HTTP Request (Twilio)           | Send typing indicator                                | HTTP Request7             | -                         |                                                   |
| HTTP Request24          | HTTP Request (Twilio)           | Send typing indicator                                | Update row in sheet5      | -                         |                                                   |
| HTTP Request28 (Twilio) | HTTP Request (Twilio)           | Send typing indicator                                | Append row in sheet2      | -                         |                                                   |
| HTTP Request17          | HTTP Request (Twilio)           | Send typing indicator                                | HTTP Request29            | -                         |                                                   |
| Google Gemini Chat Model5 | LangChain LLM Model           | AI model for data extraction                          | -                        | Basic LLM Chain3           |                                                   |
| Google Gemini Chat Model6 | LangChain LLM Model           | AI model for structured output parsing                | -                        | Structured Output Parser3  |                                                   |
| Structured Output Parser3 | LangChain Output Parser       | Parse AI output to structured JSON                    | Google Gemini Chat Model6 | Basic LLM Chain3           |                                                   |
| OpenRouter Chat Model   | LangChain LLM Model            | Fallback AI model for extraction                       | -                        | Basic LLM Chain3           |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Webhook3):**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., "5c87f92e-eba8-4f8e-825e-73a0bce05186")  
   - Purpose: Receive WhatsApp messages from Twilio webhook

2. **Create Switch Node (Switch1):**  
   - Input: Webhook3 output  
   - Conditions (on `$json.body.ButtonPayload || 'none'`):  
       - Equals "1" → output 1  
       - Equals "2" → output 2  
       - Else → default ("none")  
   - Purpose: Route based on user quick reply

3. **Create Twilio HTTP Request Nodes for Replies:**  
   - HTTP Request20 (Yes confirmation): POST to Twilio Messages API, send "Thank you for confirming!"  
   - HTTP Request21 (No confirmation): POST to Twilio Messages API, send "Please re-upload the receipt file."  
   - HTTP Request7 (Fallback): POST to Twilio Messages API, send default message  
   - Configure Twilio OAuth2 credentials for all

4. **Create Google Sheets Append Node (Append row in sheet2):**  
   - Document: Link to your Google Sheets doc  
   - Sheet: Sheet1 (or appropriate)  
   - Append MessageSid from webhook data  
   - Credentials: Google Sheets OAuth2

5. **Create HTTP Request Node to Download Media (HTTP Request5):**  
   - URL: Use `{{$json.body.MediaUrl0}}` from webhook  
   - Auth: Twilio OAuth2  
   - Timeout: 180s  
   - Retry enabled with 2s interval

6. **Create Google Drive Upload Node (Upload file2):**  
   - Name: `{{$json.body.MessageSid}}{{$binary.data.fileName}}`  
   - Folder: My Drive (or specific folder)  
   - Credentials: Google Drive OAuth2

7. **Create Google Drive Share Node (Share file1):**  
   - File ID: From Upload file2  
   - Permission: Reader, Anyone

8. **Create Google Sheets Update Node (Update row in sheet4):**  
   - Match row by MessageSid  
   - Update with Google Drive webViewLink from Share file1  
   - Credentials: Google Sheets OAuth2

9. **Create HTTP Request Node to Upload File to LlamaParse (HTTP Request29):**  
   - Method: POST  
   - URL: LlamaParse upload endpoint  
   - Auth: HTTP Bearer token (Placid credential)  
   - Body: multipart/form-data with file binary and parameters (parse_mode, model, preset "invoice", high_res_ocr, etc.)

10. **Create HTTP Request Node to Poll LlamaParse Job Status (HTTP Request28):**  
    - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{$json.id}}`  
    - Auth: Same as upload  
    - Poll repeatedly with Wait node

11. **Create Wait Node:**  
    - Wait 2 seconds between polls

12. **Create If Node (If5):**  
    - Condition: `$json.status == "SUCCESS"`  
    - True: Proceed to fetch result  
    - False: Loop back to Wait and HTTP Request28

13. **Create HTTP Request Node to Fetch OCR Result (HTTP Request30):**  
    - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{$json.id}}/result/markdown`  
    - Auth: Same as above

14. **Create LangChain Basic LLM Chain Node (Basic LLM Chain3):**  
    - Input: OCR markdown text from HTTP Request30  
    - Prompt: Use detailed receipt extraction prompt with rules (as in the workflow)  
    - Language Models: Google Gemini (primary), OpenRouter Claude (fallback)  
    - Output Parser: Structured Output Parser3

15. **Create Structured Output Parser Node (Structured Output Parser3):**  
    - Schema: JSON with fields vendor, cost, Tax, date, currency, ABN  
    - AutoFix: Enabled  

16. **Create Google Gemini Chat Model Nodes (Google Gemini Chat Model5 & 6):**  
    - Configure with Google Palm API credentials  
    - ModelName: "models/gemini-2.5-pro"

17. **Create OpenRouter Chat Model Node:**  
    - Configure with OpenRouter API key  
    - Model: "anthropic/claude-sonnet-4.5"

18. **Create Code Node for Date Normalization (Code4):**  
    - JavaScript code to normalize date formats and extract fields from AI output

19. **Create If Node (If):**  
    - Condition checks if Tax field contains "cost" or "found" (indicating missing or ambiguous tax)

20. **Create Code Node for Tax Calculation (Code5):**  
    - Computes tax as cost/11 if tax missing

21. **Create Merge Node (Merge2):**  
    - Merge normalized data and calculated tax data

22. **Create Google Sheets Update Node (Update row in sheet5):**  
    - Update row matching MessageSid with extracted data and status "Not confirmed yet"

23. **Create HTTP Request Node to Send Extracted Data via Twilio WhatsApp (HTTP Request6):**  
    - Send message with extracted vendor, cost, tax, date, ABN  
    - Use Twilio OAuth2 credential

24. **Create HTTP Request Node for Follow-up Message (HTTP Request19):**  
    - Send templated message to user

25. **Create Google Sheets Update Node (Update row in sheet11):**  
    - Update row with phone number and quick reply ID for tracking

26. **Create Merge Node (Merge3):**  
    - Merge results for final sheet updates

27. **Create Additional Google Sheets Update Nodes (Update row in sheet9, sheet10):**  
    - Update based on user confirmation (Yes/No) quick replies with status "YES" or "NO"

28. **Create HTTP Request Nodes for Typing Indicators (HTTP Request7, 17, 22, 24, 28):**  
    - Send typing indicators to user during processing to improve UX  
    - Use Twilio OAuth2 credentials  
    - Enable retries

29. **Configure all credentials:**  
    - Twilio API OAuth2 (with WhatsApp messaging enabled)  
    - Google Drive OAuth2  
    - Google Sheets OAuth2  
    - LlamaParse HTTP Bearer token (Placid)  
    - Google Palm API for Gemini  
    - OpenRouter API key

30. **Connect nodes according to the workflow diagram:**  
    - Webhook3 → Switch1 → Twilio messages or processing  
    - Receipt processing chain: Append row → download media → upload file → LlamaParse upload → poll status → fetch result → AI extraction → normalize → calculate tax → store in sheets → notify user

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Template shows automation of receipt/invoice data extraction from WhatsApp submissions using OCR + AI. Supports JPG, PNG, PDF. Utilize Twilio webhook with WhatsApp-enabled number, LlamaParse OCR, Google Gemini AI, Google Drive, and Google Sheets. Provides user prompt with quick reply buttons for confirmation.                                                                                                                                                                                                                                                                                                                                  | Sticky Note7 (Workflow intro and "Try It Out!")                    |
| Quick reply template with Yes/No buttons must be defined and approved in Twilio for WhatsApp before use.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note (Send quick reply instructions)                        |
| OCR polling mechanism uses repeated API calls to LlamaParse with 2s wait intervals to check job status before fetching results.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note1 (OCR pooling mechanism)                               |
| Tax calculation applies if tax info missing or ambiguous in extracted data, using standard Australian GST formula (tax = cost/11).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note4 (Calculate tax if not specified)                      |
| Workflow depends on multiple external APIs: Twilio (WhatsApp), LlamaParse OCR, Google Gemini (PaLM), OpenRouter (Claude model), Google Drive & Sheets. Proper API keys and OAuth2 credentials must be configured in n8n.                                                                                                                                                                                                                                                                                                                                                                                                                          | Workflow prerequisites                                              |
| Support and examples for n8n workflows can be found at the official n8n Community: https://community.n8n.io                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | https://community.n8n.io                                           |

---

**Disclaimer:** The text provided is generated exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.