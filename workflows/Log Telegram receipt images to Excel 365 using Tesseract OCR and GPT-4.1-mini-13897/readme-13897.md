Log Telegram receipt images to Excel 365 using Tesseract OCR and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/log-telegram-receipt-images-to-excel-365-using-tesseract-ocr-and-gpt-4-1-mini-13897


# Log Telegram receipt images to Excel 365 using Tesseract OCR and GPT-4.1-mini

## 1. Workflow Overview

This workflow captures receipt or payment-proof images sent to a Telegram bot, extracts text with OCR, uses an AI model to convert the text into structured transaction data, validates the result, checks for duplicates in Excel 365, and appends valid transactions to an Excel table. It then sends a Telegram response indicating success, invalid input, or a possible duplicate.

Typical use cases include small-business bookkeeping, personal expense logging, finance administration, and simple receipt-based transaction capture without manual re-entry.

### 1.1 Input Reception from Telegram
The workflow starts when a Telegram bot receives a message. It extracts the chat ID and the image file ID, then checks whether an image was actually included.

### 1.2 Image Retrieval and OCR
If an image exists, the workflow downloads the Telegram file and runs OCR with Tesseract to extract raw text from the receipt image.

### 1.3 AI-Based Structuring
The OCR text is passed to an AI Agent using `gpt-4.1-mini` as the connected chat model. The agent returns strict JSON describing the transaction fields.

### 1.4 Parsing, Configuration Loading, and Normalization
The AI JSON is parsed into workflow fields. In parallel, configuration rows are read from Excel. The transaction data is then normalized for later validation and duplicate handling.

### 1.5 Validation and Duplicate Check
The workflow verifies that core fields such as `transaction_type` and `total_amount` exist, then searches Excel for an existing matching transaction using a duplicate key.

### 1.6 Persistence and User Notification
If the transaction is valid and not a duplicate, it is saved to Excel and a Telegram confirmation is sent. Invalid or duplicate cases are also reported back to the Telegram user.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception from Telegram

**Overview:**  
This block receives inbound Telegram messages, extracts identifiers needed for downstream processing, and rejects messages that do not contain a photo. It is the main workflow entry point.

**Nodes Involved:**  
- Telegram Trigger  
- Extract Telegram chat and file ID  
- Check if image was uploaded  
- Send Notification  

### Node Details

#### Telegram Trigger
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`; webhook-based trigger for incoming Telegram bot updates.
- **Configuration choices:** Configured to listen for `message` updates only.
- **Key expressions or variables used:** None directly in node parameters beyond trigger payload usage downstream.
- **Input and output connections:** Entry point; outputs to **Extract Telegram chat and file ID**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Telegram credential or bot token invalid
  - Webhook registration issues
  - Bot not permitted to receive messages
  - Message payload does not include a `photo` array
- **Sub-workflow reference:** None.

#### Extract Telegram chat and file ID
- **Type and technical role:** `n8n-nodes-base.set`; extracts relevant values from the Telegram payload.
- **Configuration choices:** Creates:
  - `chatID = {{$json.message.chat.id}}`
  - `Image = {{$json["message"]["photo"][$json["message"]["photo"].length - 1]["file_id"]}}`
  It selects the last photo variant, which is typically the highest resolution image Telegram provides.
- **Key expressions or variables used:**
  - `message.chat.id`
  - `message.photo[length - 1].file_id`
- **Input and output connections:** Input from **Telegram Trigger**; output to **Check if image was uploaded**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If `message.photo` is missing, the expression can fail before the IF node checks emptiness
  - Non-image messages may cause expression-resolution errors depending on execution behavior
- **Sub-workflow reference:** None.

#### Check if image was uploaded
- **Type and technical role:** `n8n-nodes-base.if`; conditional gate to determine whether the image/file ID is empty.
- **Configuration choices:** Checks whether `{{$json.Image}}` is empty.
  - True branch: no image present
  - False branch: image present
- **Key expressions or variables used:** `{{$json.Image}}`
- **Input and output connections:** Input from **Extract Telegram chat and file ID**.
  - True output → **Send Notification**
  - False output → **Telegram get file**
- **Version-specific requirements:** Type version `2.2`, condition format version 2.
- **Edge cases or potential failure types:**
  - Upstream failure if the `Image` field was never created successfully
  - Empty-string logic does not protect against malformed non-empty IDs
- **Sub-workflow reference:** None.

#### Send Notification
- **Type and technical role:** `n8n-nodes-base.telegram`; sends a Telegram message when no attachment was provided.
- **Configuration choices:** Sends text `Attachment required` to `{{ $('Telegram Trigger').item.json.message.from.id }}`.
- **Key expressions or variables used:** Telegram sender ID from trigger payload.
- **Input and output connections:** Input from the true branch of **Check if image was uploaded**; no downstream nodes.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Credential mismatch: this node uses a different Telegram credential than several other Telegram nodes
  - If original sender data is missing, chat ID expression fails
- **Sub-workflow reference:** None.

---

## 2.2 Image Retrieval and OCR

**Overview:**  
This block downloads the Telegram image file and extracts its text content using Tesseract OCR. The OCR result becomes the raw input for AI-based structuring.

**Nodes Involved:**  
- Telegram get file  
- Tesseract OCR  

### Node Details

#### Telegram get file
- **Type and technical role:** `n8n-nodes-base.telegram`; retrieves the file referenced by the Telegram `file_id`.
- **Configuration choices:** Uses resource `file` with:
  - `fileId = {{ $json.Image.replace(/\n/g, '') }}`
  This strips line breaks from the file ID before retrieval.
- **Key expressions or variables used:** `{{$json.Image.replace(/\n/g, '')}}`
- **Input and output connections:** Input from false branch of **Check if image was uploaded**; output to **Tesseract OCR**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Invalid or expired Telegram file ID
  - Permission issues on bot
  - Binary download problems
  - Unexpected newline cleanup masking malformed IDs rather than rejecting them
- **Sub-workflow reference:** None.

#### Tesseract OCR
- **Type and technical role:** `n8n-nodes-tesseractjs.tesseractNode`; OCR processor converting the image binary into text.
- **Configuration choices:** Uses default options only.
- **Key expressions or variables used:** None shown explicitly.
- **Input and output connections:** Input from **Telegram get file**; output to **Parse receipt data with AI**.
- **Version-specific requirements:** Type version `1`; requires the Tesseract community node and, for some self-hosted environments, proper OCR runtime support.
- **Edge cases or potential failure types:**
  - Poor image quality leading to inaccurate text
  - Unsupported or corrupted image data
  - Missing OCR dependencies in self-hosted environments
  - Long OCR runtime on large images
- **Sub-workflow reference:** None.

---

## 2.3 AI-Based Structuring

**Overview:**  
This block converts noisy OCR text into structured transaction data. It uses an AI Agent with a connected OpenAI Chat Model and a strict system prompt designed for Indonesian financial text.

**Nodes Involved:**  
- Parse receipt data with AI  
- OpenAI Chat Model  

### Node Details

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model provider for the agent.
- **Configuration choices:** Uses model `gpt-4.1-mini` with default options.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected via AI language model port to **Parse receipt data with AI**.
- **Version-specific requirements:** Type version `1.3`; requires LangChain-compatible n8n AI nodes and valid OpenAI credentials.
- **Edge cases or potential failure types:**
  - Invalid API key
  - Rate limiting or quota exhaustion
  - Model availability changes
  - Network/API timeout
- **Sub-workflow reference:** None.

#### Parse receipt data with AI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI Agent that receives OCR text and returns structured JSON.
- **Configuration choices:**
  - Prompt text: `Here is the OCR text:\n\n{{ $json.text }}`
  - Prompt type: defined prompt
  - Detailed system message instructing:
    - transaction classification (`TRANSFER`, `PURCHASE`, `EWALLET`, `OTHER`)
    - extraction rules for sender, recipient, merchant, amount, fee, total, datetime, reference, description
    - validation constraints
    - strict JSON-only output
- **Key expressions or variables used:** `{{$json.text}}`
- **Input and output connections:** Input from **Tesseract OCR**; output to **JSON output parsing**. Receives model input from **OpenAI Chat Model**.
- **Version-specific requirements:** Type version `3`; depends on installed AI/LangChain support in n8n.
- **Edge cases or potential failure types:**
  - OCR text may be too poor for reliable extraction
  - LLM may still return malformed JSON despite instructions
  - Ambiguous receipt formats
  - Hallucination risk reduced but not eliminated
- **Sub-workflow reference:** None.

---

## 2.4 Parsing, Configuration Loading, and Normalization

**Overview:**  
This block turns the AI output into workflow-accessible JSON, loads configuration values from Excel, and computes normalized fields used for validation and duplicate detection.

**Nodes Involved:**  
- JSON output parsing  
- Get Configurations  
- Normalize transaction data  

### Node Details

#### JSON output parsing
- **Type and technical role:** `n8n-nodes-base.set`; converts the agent’s output string into raw JSON output for downstream usage.
- **Configuration choices:** Uses raw mode with `jsonOutput = {{ $json.output }}`.
- **Key expressions or variables used:** `{{$json.output}}`
- **Input and output connections:** Input from **Parse receipt data with AI**; outputs to both **Normalize transaction data** and **Get Configurations**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If AI output is not valid JSON, parsing can fail
  - Missing `output` field from the agent
- **Sub-workflow reference:** None.

#### Get Configurations
- **Type and technical role:** `n8n-nodes-base.microsoftExcel`; reads configuration rows from an Excel table.
- **Configuration choices:**
  - Resource: `table`
  - Operation: `getRows`
  - Return all rows: enabled
  - Reads from workbook `Contoh Excel N8N`, worksheet `Smart Expense Guardian`, table `Configuration`
- **Key expressions or variables used:** None inside the node itself; values are later accessed via `.all().find(...)`.
- **Input and output connections:** Input from **JSON output parsing**; output to **Normalize transaction data**.
- **Version-specific requirements:** Type version `2.2`; requires Microsoft Excel OAuth2 credentials and file/table availability in OneDrive or SharePoint.
- **Edge cases or potential failure types:**
  - OAuth token expiration
  - Workbook/table renamed or moved
  - Configuration table missing expected `Key`/`Value` columns
- **Sub-workflow reference:** None.

#### Normalize transaction data
- **Type and technical role:** `n8n-nodes-base.set`; computes normalized text and loads configuration values into the current item.
- **Configuration choices:** Adds fields including:
  - `normalized_total`
  - `normalized_sender_name`
  - `normalized_merchant_name`
  - `LIMIT_AMOUNT_REVIEW`
  - `CHAT_ID_APPROVER`
  - `ENABLE_DUPLICATE_CHECK`
  - `DUPLICATE_STRATEGY`
  - `DEFAULT_TIMEZONE`
  - `AUTO_APPEND`
  - `REQUIRE_REFERENCE`
  - `REVIEW_MODE`
  It includes other fields from prior nodes as well.
- **Key expressions or variables used:**
  - `parseFloat($('JSON output parsing').first().json.total_amount.toString().replace(/[^0-9.]/g, ""))`
  - `$('JSON output parsing').first().json.sender_name.toUpperCase().replace(/\s+/g,'')`
  - `$('JSON output parsing').first().json.merchant_name.toUpperCase().replace(/\s+/g,'')`
  - Multiple lookups from `$('Get Configurations').all().find(i => i.json.Key === "...")?.json.Value`
- **Input and output connections:** Receives data after **Get Configurations**; outputs to **Validate required transaction fields**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - `sender_name` or `merchant_name` being null causes `.toUpperCase()` failure
  - `total_amount` null causes `.toString()` failure
  - `DEFAULT_TIMEZONE` is incorrectly populated from the `DUPLICATE_STRATEGY` key instead of `DEFAULT_TIMEZONE`
  - Several loaded config values are not actually used later
- **Sub-workflow reference:** None.

---

## 2.5 Validation and Duplicate Check

**Overview:**  
This block ensures that required transaction fields exist and then checks Excel for a matching duplicate entry. It protects the final Excel table from incomplete or repeated records.

**Nodes Involved:**  
- Validate required transaction fields  
- Send Notification image not valid  
- Check existing transaction in Excel  
- Check for duplicate transaction  
- Send notification duplicate  

### Node Details

#### Validate required transaction fields
- **Type and technical role:** `n8n-nodes-base.if`; validates presence of critical fields.
- **Configuration choices:** Requires:
  - `parseInt($('JSON output parsing').first().total_amount)` to be not empty
  - `$('JSON output parsing').first().json.transaction_type` to be not empty
- **Key expressions or variables used:**
  - `={{ parseInt($('JSON output parsing').first().total_amount) }}`
  - `={{ $('JSON output parsing').first().json.transaction_type }}`
- **Input and output connections:** Input from **Normalize transaction data**.
  - True output → **Check existing transaction in Excel**
  - False output → **Send Notification image not valid**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - The first expression appears inconsistent: `.first().total_amount` should likely be `.first().json.total_amount`
  - If malformed, the expression may always fail or produce undefined
  - Validation is minimal and does not check merchant/sender/date consistency
- **Sub-workflow reference:** None.

#### Send Notification image not valid
- **Type and technical role:** `n8n-nodes-base.telegram`; notifies the user that extraction/validation failed.
- **Configuration choices:** Sends `Receipt could not be validated. Please resend.` to the original Telegram sender.
- **Key expressions or variables used:** `{{ $('Telegram Trigger').item.json.message.from.id }}`
- **Input and output connections:** Input from false branch of **Validate required transaction fields**; no downstream nodes.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Telegram credential problems
  - Missing sender ID in trigger data
- **Sub-workflow reference:** None.

#### Check existing transaction in Excel
- **Type and technical role:** `n8n-nodes-base.microsoftExcel`; looks up an existing row by duplicate key.
- **Configuration choices:**
  - Resource: `table`
  - Operation: `lookup`
  - `returnAllMatches = true`
  - Workbook: `Contoh Excel N8N`
  - Worksheet: `Smart Expense Guardian`
  - Table: `FinanceIntake`
  - Lookup column: `Duplicate Key`
  - Lookup value: `{{ $json.normalized_vendor + "_" + $json.date + "_" + $json.normalized_total }}`
- **Key expressions or variables used:** `{{$json.normalized_vendor + "_" + $json.date + "_" + $json.normalized_total}}`
- **Input and output connections:** Input from true branch of **Validate required transaction fields**; output to **Check for duplicate transaction**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - `normalized_vendor` and `date` are not created anywhere in this workflow, so the lookup key expression is likely broken
  - OAuth or Excel table access failures
  - Lookup column naming mismatch
- **Sub-workflow reference:** None.

#### Check for duplicate transaction
- **Type and technical role:** `n8n-nodes-base.if`; decides whether any matching Excel rows were found.
- **Configuration choices:** Checks whether `parseInt($json.length)>0` is true.
- **Key expressions or variables used:** `={{ parseInt($json.length)>0 }}`
- **Input and output connections:** Input from **Check existing transaction in Excel**.
  - True output → **Send notification duplicate**
  - False output → **Save transaction to Excel** and **Send confirmation message**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Excel lookup output may not expose data as `length` on `$json`; this depends on node response shape
  - Duplicate-detection logic may therefore misfire
  - Branching sends success message in parallel with save, so message may still go out if save later fails
- **Sub-workflow reference:** None.

#### Send notification duplicate
- **Type and technical role:** `n8n-nodes-base.telegram`; sends a duplicate warning to the user.
- **Configuration choices:** Sends `Possible duplicate receipt detected.` to the Telegram sender.
- **Key expressions or variables used:** `{{ $('Telegram Trigger').item.json.message.from.id }}`
- **Input and output connections:** Input from true branch of **Check for duplicate transaction**; no downstream nodes.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Telegram credential issues
  - False positives caused by broken duplicate-key logic
- **Sub-workflow reference:** None.

---

## 2.6 Persistence and User Notification

**Overview:**  
This block writes approved transactions to Excel and notifies the Telegram user that the receipt was recorded. It is the final persistence step of the workflow.

**Nodes Involved:**  
- Save transaction to Excel  
- Send confirmation message  

### Node Details

#### Save transaction to Excel
- **Type and technical role:** `n8n-nodes-base.microsoftExcel`; appends a row to the target transactions table.
- **Configuration choices:** Writes to table `FinanceIntake` in workbook `Contoh Excel N8N`, worksheet `Smart Expense Guardian`. Mapped fields include sender/recipient details, merchant, type, amounts, reference, description, timestamps, status, and duplicate key.
- **Key expressions or variables used:**
  - Submitted By: `{{ $('Telegram Trigger').item.json.message.from.first_name }}`
  - Duplicate Key:
    - For transfer: `transaction_datetime + total_amount + recipient_name`
    - Otherwise: `transaction_datetime + total_amount + merchant_name`
  - Status: `{{ $('Validate required transaction fields').item.json.normalized_total>=$('Get Configurations').item.json.Value ? 'Pending' : 'Approved' }}`
  - Submission ID: `{{$execution.id}}`
  - Created: `{{$now}}`
- **Input and output connections:** Input from false branch of **Check for duplicate transaction**; no downstream nodes.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Status expression likely incorrect because `$('Get Configurations').item.json.Value` is ambiguous when multiple config rows exist
  - Duplicate key logic here differs from the lookup logic, so duplicate prevention is inconsistent
  - Excel append can fail because of workbook locks, auth issues, or missing table columns
- **Sub-workflow reference:** None.

#### Send confirmation message
- **Type and technical role:** `n8n-nodes-base.telegram`; confirms successful logging to the Telegram user.
- **Configuration choices:** Sends `Receipt recorded successfully.` to the sender.
- **Key expressions or variables used:** `{{ $('Telegram Trigger').item.json.message.from.id }}`
- **Input and output connections:** Input from false branch of **Check for duplicate transaction**; no downstream nodes.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Sent in parallel with Excel save, so a user may receive success even if the save node fails afterward
  - Telegram API or credential failures
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Receives incoming Telegram messages |  | Extract Telegram chat and file ID | ## Telegram trigger<br>Receive receipt images from a Telegram bot and extract chat and file information for further processing. |
| Extract Telegram chat and file ID | n8n-nodes-base.set | Extracts chat ID and highest-resolution photo file ID | Telegram Trigger | Check if image was uploaded | ## Telegram trigger<br>Receive receipt images from a Telegram bot and extract chat and file information for further processing. |
| Check if image was uploaded | n8n-nodes-base.if | Verifies that the incoming Telegram message contains an image | Extract Telegram chat and file ID | Send Notification; Telegram get file | ## Telegram trigger<br>Receive receipt images from a Telegram bot and extract chat and file information for further processing. |
| Send Notification | n8n-nodes-base.telegram | Sends error message if no attachment is present | Check if image was uploaded |  |  |
| Telegram get file | n8n-nodes-base.telegram | Downloads the Telegram file by file ID | Check if image was uploaded | Tesseract OCR | ## Image processing and OCR<br>Download the image and extract text content from the receipt using OCR. |
| Tesseract OCR | n8n-nodes-tesseractjs.tesseractNode | Performs OCR on the downloaded image | Telegram get file | Parse receipt data with AI | ## Image processing and OCR<br>Download the image and extract text content from the receipt using OCR. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the LLM backend for the AI agent |  | Parse receipt data with AI | ## AI data extraction<br>Convert raw OCR text into structured transaction data using an AI model. |
| Parse receipt data with AI | @n8n/n8n-nodes-langchain.agent | Converts OCR text into structured transaction JSON | Tesseract OCR; OpenAI Chat Model | JSON output parsing | ## AI data extraction<br>Convert raw OCR text into structured transaction data using an AI model. |
| JSON output parsing | n8n-nodes-base.set | Parses the AI output into JSON fields | Parse receipt data with AI | Normalize transaction data; Get Configurations | ## AI data extraction<br>Convert raw OCR text into structured transaction data using an AI model. |
| Get Configurations | n8n-nodes-base.microsoftExcel | Loads configuration rows from Excel | JSON output parsing | Normalize transaction data | ## Data validation and duplicate check<br>Validate required transaction fields and prevent duplicate records. |
| Normalize transaction data | n8n-nodes-base.set | Normalizes values and loads config values into the item | JSON output parsing; Get Configurations | Validate required transaction fields | ## Data validation and duplicate check<br>Validate required transaction fields and prevent duplicate records. |
| Validate required transaction fields | n8n-nodes-base.if | Checks core extracted fields before persistence | Normalize transaction data | Check existing transaction in Excel; Send Notification image not valid | ## Data validation and duplicate check<br>Validate required transaction fields and prevent duplicate records. |
| Send Notification image not valid | n8n-nodes-base.telegram | Informs user that receipt validation failed | Validate required transaction fields |  | ## Data validation and duplicate check<br>Validate required transaction fields and prevent duplicate records. |
| Check existing transaction in Excel | n8n-nodes-base.microsoftExcel | Searches Excel for a duplicate transaction | Validate required transaction fields | Check for duplicate transaction | ## Data validation and duplicate check<br>Validate required transaction fields and prevent duplicate records. |
| Check for duplicate transaction | n8n-nodes-base.if | Branches based on whether a duplicate exists | Check existing transaction in Excel | Send notification duplicate; Save transaction to Excel; Send confirmation message | ## Data validation and duplicate check<br>Validate required transaction fields and prevent duplicate records. |
| Send notification duplicate | n8n-nodes-base.telegram | Warns user about a possible duplicate | Check for duplicate transaction |  | ## Data validation and duplicate check<br>Validate required transaction fields and prevent duplicate records. |
| Save transaction to Excel | n8n-nodes-base.microsoftExcel | Appends the validated transaction to Excel | Check for duplicate transaction |  | ## Save transaction and notify user<br>Store the transaction in Excel and send a confirmation message to Telegram. |
| Send confirmation message | n8n-nodes-base.telegram | Sends success confirmation to the user | Check for duplicate transaction |  | ## Save transaction and notify user<br>Store the transaction in Excel and send a confirmation message to Telegram. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for Telegram trigger area |  |  | ## Telegram trigger<br>Receive receipt images from a Telegram bot and extract chat and file information for further processing. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for OCR area |  |  | ## Image processing and OCR<br>Download the image and extract text content from the receipt using OCR. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for validation area |  |  | ## Data validation and duplicate check<br>Validate required transaction fields and prevent duplicate records. |
| Sticky Note3 | n8n-nodes-base.stickyNote | General overview and requirements note |  |  | # 📸💰 Log receipt transactions from Telegram images to Excel using OCR and AI<br>OCR + AI + Duplicate Protection (n8n Workflow)<br>⸻<br>## Overview<br>This workflow automatically converts receipt images or payment proof sent via Telegram into structured financial records stored in Excel 365.<br>Core flow:<br>`Take Photo → OCR → AI Structuring → Validation → Duplicate Check → Append to Excel`<br>The system is designed to reduce manual data entry errors and improve financial visibility by turning unstructured receipt images into reliable financial records.<br>⸻<br>## 🧠 What This Workflow Does<br>This workflow automatically:<br>- Captures receipt images from Telegram<br>- Extracts text using OCR<br>- Converts OCR text into structured transaction data using AI<br>- Validates required financial fields<br>- Prevents duplicate transaction entries<br>- Saves valid records into Excel<br>- Sends confirmation messages to Telegram<br>⸻<br>## 🎯 Key Benefits<br>• Reduce financial recording errors<br>• Eliminate repetitive manual data entry<br>• Improve cashflow visibility<br>• Simple photo-based financial recording<br>• Scalable foundation for AI-powered financial automation<br>⸻<br>## 🚀 Use Cases<br>This workflow is useful for:<br>• Small business owners<br>• Online sellers<br>• Freelancers<br>• Finance administrators<br>• Personal expense tracking<br>⸻<br>## 📌 Notes<br>• Duplicate detection is based on a composite transaction key<br>• OCR accuracy depends on image quality<br>• AI parsing improves recognition compared to OCR-only approaches<br>⸻<br>## 🧱 Requirements<br>To run this workflow you need:<br>• n8n (Cloud or Self-Hosted)<br>• Telegram Bot Token<br>• Microsoft Excel 365 Account<br>• OpenAI API Key or Google Gemini API Key<br>• Tesseract OCR (for self-hosted installations)<br>⸻<br>## 🏁 Result<br>After setup, users can simply send receipt photos to Telegram and the workflow will automatically convert them into structured financial records stored in Excel.<br>No manual bookkeeping required.<br>⸻<br>Built for automation lovers, small businesses, and teams who want simple but powerful financial tracking using n8n. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Lists workflow nodes |  |  | ## Workflow Nodes<br>The workflow contains the following nodes:<br>1. Telegram Trigger<br>1. Edit Fields (extract chat_id and image_id)<br>1. Condition (validate image exists)<br>1. Telegram Get File<br>1. Tesseract OCR<br>1. AI Agent (ChatGPT / Gemini)<br>1. Edit Field (JSON Output Parsing)<br>1. Excel 365 – Get Configuration<br>1. Edit Field – Normalize & Generate duplicate_key<br>1. Condition – Validate required fields<br>1. Excel 365 – Lookup duplicate_key<br>1. Condition – Duplicate Check<br>1. Excel 365 – Append Transaction<br>1. Telegram – Send Confirmation Message |
| Sticky Note5 | n8n-nodes-base.stickyNote | Setup note |  |  | ## Setup steps<br>1. Create a Telegram bot using @BotFather and copy the Bot Token.<br>2. Add Telegram credentials in n8n using the Bot Token.<br>3. Prepare an Excel file in OneDrive or SharePoint with a TRANSACTIONS sheet.<br>4. Configure Microsoft Excel credentials in n8n.<br>5. Add either OpenAI or Google Gemini credentials for the AI parsing step.<br>6. If running self-hosted n8n, ensure Tesseract OCR is installed.<br>7. Activate the workflow and send a receipt image to the Telegram bot to test. |
| Sticky Note7 | n8n-nodes-base.stickyNote | High-level operating explanation |  |  | ## How it works<br>This workflow captures receipt images sent to a Telegram bot and converts them into structured financial records in Excel.<br><br>When a user sends an image, the workflow downloads the file and extracts the text using OCR. The extracted text is then analyzed by an AI model (OpenAI or Gemini) to identify key transaction details such as date, amount, merchant, and transaction type.<br><br>The workflow validates the extracted data and generates a unique duplicate key. It then checks Excel to ensure the transaction has not already been recorded. If no duplicate is found, the transaction is appended to the Excel sheet.<br><br>The user receives a confirmation message in Telegram indicating whether the transaction was successfully recorded or rejected due to missing data or duplication. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation note for AI extraction area |  |  | ## AI data extraction<br>Convert raw OCR text into structured transaction data using an AI model. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note for save/notify area |  |  | ## Save transaction and notify user<br>Store the transaction in Excel and send a confirmation message to Telegram. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Log receipt transactions from Telegram images to Excel using OCR and AI`.

2. **Add a Telegram Trigger node**
   - Node type: `Telegram Trigger`
   - Configure it to listen to `message` updates.
   - Connect Telegram bot credentials created from a BotFather token.
   - This is the workflow entry point.

3. **Add a Set node named `Extract Telegram chat and file ID`**
   - Create field `chatID` as string:
     - `{{$json.message.chat.id}}`
   - Create field `Image` as string:
     - `{{$json["message"]["photo"][$json["message"]["photo"].length - 1]["file_id"]}}`
   - Connect `Telegram Trigger -> Extract Telegram chat and file ID`.

4. **Add an IF node named `Check if image was uploaded`**
   - Condition: check whether `{{$json.Image}}` is empty.
   - True branch = no image present.
   - False branch = image exists.
   - Connect `Extract Telegram chat and file ID -> Check if image was uploaded`.

5. **Add a Telegram node named `Send Notification`**
   - Operation: send message
   - Chat ID:
     - `{{ $('Telegram Trigger').item.json.message.from.id }}`
   - Text:
     - `Attachment required`
   - Connect the true output of `Check if image was uploaded` to this node.

6. **Add a Telegram node named `Telegram get file`**
   - Resource: `file`
   - File ID:
     - `{{ $json.Image.replace(/\n/g, '') }}`
   - Connect the false output of `Check if image was uploaded` to this node.
   - Use Telegram credentials with permission to access the bot file.

7. **Add a Tesseract OCR node named `Tesseract OCR`**
   - Use default OCR settings unless you need language-specific tuning.
   - Connect `Telegram get file -> Tesseract OCR`.
   - If self-hosted, ensure Tesseract or the node’s OCR dependency is installed and functioning.

8. **Add an OpenAI Chat Model node**
   - Node type: `OpenAI Chat Model`
   - Choose model: `gpt-4.1-mini`
   - Add OpenAI API credentials.
   - Leave advanced options at defaults unless needed.

9. **Add an AI Agent node named `Parse receipt data with AI`**
   - Prompt text:
     - `Here is the OCR text:\n\n{{ $json.text }}`
   - Prompt type: define prompt
   - Add the system message from the workflow, which instructs the model to:
     - classify transaction type
     - extract sender/recipient/bank/merchant/amount/fee/total/datetime/reference/description
     - return JSON only
   - Connect `Tesseract OCR -> Parse receipt data with AI`.
   - Connect `OpenAI Chat Model` to the AI language model input of this node.

10. **Add a Set node named `JSON output parsing`**
    - Mode: raw
    - JSON output:
      - `{{ $json.output }}`
    - Connect `Parse receipt data with AI -> JSON output parsing`.
    - This assumes the agent returns valid JSON in `output`.

11. **Prepare your Excel workbook**
    - Store it in OneDrive or SharePoint.
    - Create at least:
      - A configuration table, e.g. `Configuration`
      - A transaction table, e.g. `FinanceIntake`
    - Expected transaction columns used by this workflow:
      - Sender Name
      - Sender Bank
      - Recipient Name
      - Recipient Bank
      - Merchant Name
      - Transaction Type
      - Submitted By
      - Duplicate Key
      - Status
      - Submission ID
      - Created
      - Amount
      - Fee
      - Total Amount
      - Transaction Date
      - Reference Number
      - Description
    - Expected configuration columns:
      - `Key`
      - `Value`

12. **Add a Microsoft Excel node named `Get Configurations`**
    - Resource: `table`
    - Operation: `getRows`
    - Select the workbook, worksheet, and the `Configuration` table.
    - Enable `Return All`.
    - Connect `JSON output parsing -> Get Configurations`.
    - Add Microsoft Excel OAuth2 credentials.

13. **Add a Set node named `Normalize transaction data`**
    - Enable “include other fields”.
    - Add the following fields:
      - `normalized_total`:
        - `{{ parseFloat($('JSON output parsing').first().json.total_amount.toString().replace(/[^0-9.]/g, "")) }}`
      - `normalized_sender_name`:
        - `{{ $('JSON output parsing').first().json.sender_name.toUpperCase().replace(/\s+/g,'') }}`
      - `normalized_merchant_name`:
        - `{{ $('JSON output parsing').first().json.merchant_name.toUpperCase().replace(/\s+/g,'') }}`
      - `LIMIT_AMOUNT_REVIEW`:
        - `{{ $('Get Configurations').all().find(i => i.json.Key === "LIMIT_AMOUNT_REVIEW")?.json.Value }}`
      - `CHAT_ID_APPROVER`:
        - `{{ $('Get Configurations').all().find(i => i.json.Key === "CHAT_ID_APPROVER")?.json.Value }}`
      - `ENABLE_DUPLICATE_CHECK`:
        - `{{ $('Get Configurations').all().find(i => i.json.Key === "ENABLE_DUPLICATE_CHECK")?.json.Value }}`
      - `DUPLICATE_STRATEGY`:
        - `{{ $('Get Configurations').all().find(i => i.json.Key === "DUPLICATE_STRATEGY")?.json.Value }}`
      - `DEFAULT_TIMEZONE`:
        - In the provided workflow this incorrectly reads `DUPLICATE_STRATEGY`; you should correct it to:
        - `{{ $('Get Configurations').all().find(i => i.json.Key === "DEFAULT_TIMEZONE")?.json.Value }}`
      - `AUTO_APPEND`:
        - `{{ $('Get Configurations').all().find(i => i.json.Key === "AUTO_APPEND")?.json.Value }}`
      - `REQUIRE_REFERENCE`:
        - `{{ $('Get Configurations').all().find(i => i.json.Key === "REQUIRE_REFERENCE")?.json.Value }}`
      - `REVIEW_MODE`:
        - `{{ $('Get Configurations').all().find(i => i.json.Key === "REVIEW_MODE")?.json.Value }}`
    - Connect `Get Configurations -> Normalize transaction data`.

14. **Add an IF node named `Validate required transaction fields`**
    - Check that `total_amount` exists and `transaction_type` exists.
    - Recommended corrected expressions:
      - `{{ parseInt($('JSON output parsing').first().json.total_amount) }}`
      - `{{ $('JSON output parsing').first().json.transaction_type }}`
    - In the source workflow, the first expression is likely malformed because it omits `.json`.
    - Connect `Normalize transaction data -> Validate required transaction fields`.

15. **Add a Telegram node named `Send Notification image not valid`**
    - Chat ID:
      - `{{ $('Telegram Trigger').item.json.message.from.id }}`
    - Text:
      - `Receipt could not be validated. Please resend.`
    - Connect the false branch of `Validate required transaction fields` to this node.

16. **Add a Microsoft Excel node named `Check existing transaction in Excel`**
    - Resource: `table`
    - Operation: `lookup`
    - Select the transaction table `FinanceIntake`.
    - Lookup column: `Duplicate Key`
    - Enable `Return All Matches`
    - The original workflow uses:
      - `{{ $json.normalized_vendor + "_" + $json.date + "_" + $json.normalized_total }}`
    - This is inconsistent because `normalized_vendor` and `date` are never created.
    - Recommended fix:
      - Create a proper duplicate key field earlier, for example:
        - transfer: `transaction_datetime + "_" + normalized_total + "_" + normalized recipient/merchant`
      - use the same expression both for lookup and save.
    - Connect true branch of `Validate required transaction fields` to this node.

17. **Add an IF node named `Check for duplicate transaction`**
    - Condition should detect whether the Excel lookup returned one or more rows.
    - The source workflow uses `{{ parseInt($json.length)>0 }}`, which may not match actual node output shape.
    - Recommended approach:
      - inspect the Excel lookup output and test for returned items count
      - if lookup returns items directly, use item existence rather than `$json.length`
    - Connect `Check existing transaction in Excel -> Check for duplicate transaction`.

18. **Add a Telegram node named `Send notification duplicate`**
    - Chat ID:
      - `{{ $('Telegram Trigger').item.json.message.from.id }}`
    - Text:
      - `Possible duplicate receipt detected.`
    - Connect the true branch of `Check for duplicate transaction` to this node.

19. **Add a Microsoft Excel node named `Save transaction to Excel`**
    - Resource: `table`
    - Operation: append/add row
    - Select transaction table `FinanceIntake`
    - Map fields:
      - Sender Name = `{{ $('Validate required transaction fields').item.json.sender_name }}`
      - Sender Bank = `{{ $('Validate required transaction fields').item.json.sender_bank }}`
      - Recipient Name = `{{ $('Validate required transaction fields').item.json.recipient_name }}`
      - Recipient Bank = `{{ $('Validate required transaction fields').item.json.recipient_bank }}`
      - Merchant Name = `{{ $('Validate required transaction fields').item.json.merchant_name }}`
      - Transaction Type = `{{ $('Validate required transaction fields').item.json.transaction_type }}`
      - Submitted By = `{{ $('Telegram Trigger').item.json.message.from.first_name }}`
      - Duplicate Key = use one stable expression; the source workflow uses conditional concatenation based on transaction type
      - Status = source workflow compares against a configuration value, but the expression is ambiguous; recommended:
        - compare `normalized_total` to the config row for `LIMIT_AMOUNT_REVIEW`
      - Submission ID = `{{$execution.id}}`
      - Created = `{{$now}}`
      - Amount = `{{ $('Validate required transaction fields').item.json.amount }}`
      - Fee = `{{ $('Validate required transaction fields').item.json.fee }}`
      - Total Amount = `{{ $('Validate required transaction fields').item.json.total_amount }}`
      - Transaction Date = `{{ $('Validate required transaction fields').item.json.transaction_datetime }}`
      - Reference Number = `{{ $('Validate required transaction fields').item.json.reference_number }}`
      - Description = `{{ $('Validate required transaction fields').item.json.description }}`
    - Connect false branch of `Check for duplicate transaction` to this node.

20. **Add a Telegram node named `Send confirmation message`**
    - Chat ID:
      - `{{ $('Telegram Trigger').item.json.message.from.id }}`
    - Text:
      - `Receipt recorded successfully.`
    - Connect false branch of `Check for duplicate transaction` to this node.
    - For stronger reliability, consider sending this only after a successful Excel append rather than in parallel.

21. **Add sticky notes if desired**
    - Telegram trigger area
    - OCR area
    - AI extraction area
    - validation/duplicate-check area
    - save/notify area
    - overview/setup notes

22. **Configure credentials**
    - **Telegram**
      - Create bot with `@BotFather`
      - Add Telegram API credentials in n8n
    - **Microsoft Excel OAuth2**
      - Connect Microsoft 365 account with access to workbook in OneDrive or SharePoint
    - **OpenAI**
      - Add OpenAI API key credentials
    - **Tesseract**
      - Ensure the OCR node is available and supported in your n8n environment

23. **Test the workflow**
    - Send a real receipt photo to the Telegram bot
    - Verify:
      - OCR output contains text
      - AI returns valid JSON
      - validation branch behaves as expected
      - duplicate lookup works with your chosen duplicate key
      - Excel row is appended correctly
      - Telegram feedback matches outcome

24. **Recommended corrections before production use**
    - Guard the photo extraction expression against non-photo messages
    - Fix `Validate required transaction fields` to use `.json.total_amount`
    - Fix `DEFAULT_TIMEZONE` config lookup
    - Standardize duplicate-key generation between lookup and append
    - Ensure duplicate-check condition reflects actual Excel output
    - Move success notification after successful Excel save

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically converts receipt images or payment proof sent via Telegram into structured financial records stored in Excel 365. | Overview note from workflow |
| Core flow: Take Photo → OCR → AI Structuring → Validation → Duplicate Check → Append to Excel | Overview note from workflow |
| Key benefits: reduce financial recording errors, eliminate repetitive manual data entry, improve cashflow visibility, simple photo-based financial recording, scalable foundation for AI-powered financial automation. | Workflow overview note |
| Useful for: small business owners, online sellers, freelancers, finance administrators, personal expense tracking. | Workflow overview note |
| Notes: duplicate detection is based on a composite transaction key; OCR accuracy depends on image quality; AI parsing improves recognition compared to OCR-only approaches. | Workflow overview note |
| Requirements: n8n (Cloud or Self-Hosted), Telegram Bot Token, Microsoft Excel 365 Account, OpenAI API Key or Google Gemini API Key, Tesseract OCR for self-hosted installations. | Workflow overview note |
| Setup steps: create Telegram bot with @BotFather, configure Telegram credentials, prepare Excel workbook and tables, configure Microsoft Excel credentials, add OpenAI or Gemini credentials, ensure Tesseract is available, activate workflow and test with a receipt image. | Setup note |
| Result: users can send receipt photos to Telegram and the workflow converts them into structured financial records stored in Excel with no manual bookkeeping. | Workflow overview note |