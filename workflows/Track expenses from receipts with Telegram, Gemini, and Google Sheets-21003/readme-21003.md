Track expenses from receipts with Telegram, Gemini, and Google Sheets

https://n8nworkflows.xyz/workflows/track-expenses-from-receipts-with-telegram--gemini--and-google-sheets-21003


# Track expenses from receipts with Telegram, Gemini, and Google Sheets

### 1. Workflow Overview

The **“Track expenses from receipts with Telegram, Gemini, and Google Sheets”** workflow automates the extraction, validation, and categorization of expense data from receipts submitted via Telegram. It leverages Google Gemini for multimodal AI extraction, utilizes an n8n Data Table paired with the Wait node for human-in-the-loop validation when confidence is low, and logs finalized expense data into Google Sheets.

The logic is divided into five functional blocks:
- **1.1 Input Reception & Routing:** Listens for Telegram updates (photos, documents, text messages, or inline button callbacks) and branches execution based on input type.
- **1.2 Receipt Processing & AI Extraction:** Downloads binary files from Telegram and sends them to Google Gemini to extract structured expense metadata (vendor, amount, date, category, payment method, confidence score).
- **1.3 Validation & Automated Routing:** Parses Gemini's JSON output, normalizes dates, and evaluates the confidence score against a threshold (0.65). High-confidence items proceed directly to logging, while low-confidence items are staged for user review.
- **1.4 Human-in-the-Loop Review & Callback Handling:** Temporarily stores low-confidence expenses in an n8n Data Table, pauses execution via a Wait node, prompts the user via interactive Telegram inline buttons, and routes incoming button callbacks to resume the paused execution.
- **1.5 Data Persistence & Confirmation:** Appends approved or high-confidence expense data to Google Sheets, cleans up temporary Data Table records, and sends a final success notification back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing
- **Overview:** Acts as the primary entry point, capturing all incoming Telegram webhooks and inspecting the payload structure to route messages down distinct operational paths.
- **Nodes Involved:** 
  - `Telegram Message Trigger`
  - `Route by Telegram Input`
- **Node Details:**
  - **Telegram Message Trigger**
    - *Type/Role:* `n8n-nodes-base.telegramTrigger` (Trigger)
    - *Configuration:* Listens for `message` and `callback_query` update types.
    - *Credentials:* Telegram API (`your bot`).
    - *Connections:* Input: None; Output: `Route by Telegram Input`.
    - *Edge Cases:* Webhook delivery failure if the n8n instance is not publicly accessible.
  - **Route by Telegram Input**
    - *Type/Role:* `n8n-nodes-base.switch` (Flow Control)
    - *Configuration:* Evaluates expressions to branch output: Index 0 (`message.photo`), Index 1 (`message.document`), Index 2 (`message.text`), Index 3 (`callback_query`).
    - *Expressions:* `={{ !!$json.message.photo }}`, `={{ !!$json.message.document }}`, `={{ !!$json.message.text }}`, `={{ !!$json.callback_query }}`.
    - *Connections:* Input: `Telegram Message Trigger`; Output: Downstream download, text-handling, or callback-parsing nodes.

---

#### 2.2 Receipt Processing & AI Extraction
- **Overview:** Downloads binary receipt assets (images or documents) from Telegram's servers and submits them to Google Gemini for structural JSON extraction.
- **Nodes Involved:**
  - `Download Telegram Photo`
  - `Download Telegram Document`
  - `Handle Unsupported Text Input`
  - `Analyze Receipt Photo with Gemini`
  - `Analyze Receipt Document with Gemini`
- **Node Details:**
  - **Download Telegram Photo**
    - *Type/Role:* `n8n-nodes-base.telegram` (Action / File Download)
    - *Configuration:* Resource: `file`. Selects the highest-resolution photo from the array.
    - *Expressions:* `={{ $json.message.photo[$json.message.photo.length - 1].file_id }}`.
    - *Connections:* Input: `Route by Telegram Input` (Branch 0); Output: `Analyze Receipt Photo with Gemini`.
  - **Download Telegram Document**
    - *Type/Role:* `n8n-nodes-base.telegram` (Action / File Download)
    - *Configuration:* Resource: `file`.
    - *Expressions:* `={{ $json.message.document.file_id }}`.
    - *Connections:* Input: `Route by Telegram Input` (Branch 1); Output: `Analyze Receipt Document with Gemini`.
  - **Handle Unsupported Text Input**
    - *Type/Role:* `n8n-nodes-base.telegram` (Action / Message)
    - *Configuration:* Sends a notification stating "I only work with files and photos".
    - *Expressions:* Chat ID derived from `$('Telegram Message Trigger').item.json.message.chat.id`.
    - *Connections:* Input: `Route by Telegram Input` (Branch 2); Output: None (Terminal node).
  - **Analyze Receipt Photo with Gemini**
    - *Type/Role:* `@n8n/n8n-nodes-langchain.googleGemini` (AI / Multimodal Analysis)
    - *Configuration:* Resource: `image`, Input Type: `binary`, Operation: `analyze`. Enforces strict JSON output containing vendor, amount, date (`YYYY-MM-DD`), category (Food, Transport, Shopping, Services, Other), payment method, and confidence score. Has retry on fail enabled.
    - *Credentials:* Google Gemini (PaLM) API account.
    - *Connections:* Input: `Download Telegram Photo`; Output: `Parse Photo Analysis JSON`.
  - **Analyze Receipt Document with Gemini**
    - *Type/Role:* `@n8n/n8n-nodes-langchain.googleGemini` (AI / Multimodal Analysis)
    - *Configuration:* Resource: `document`, Input Type: `binary`. Uses the same strict JSON prompt as the photo analyzer. Has retry on fail enabled.
    - *Credentials:* Google Gemini (PaLM) API account.
    - *Connections:* Input: `Download Telegram Document`; Output: `Parse Document Analysis JSON`.

---

#### 2.3 Validation & Automated Routing
- **Overview:** Parses Gemini's textual response into structured JSON, normalizes faulty date strings, and branches workflow execution based on the model's confidence rating.
- **Nodes Involved:**
  - `Parse Photo Analysis JSON`
  - `Parse Document Analysis JSON`
  - `Check Photo Analysis Confidence`
  - `Check Document Analysis Confidence`
- **Node Details:**
  - **Parse Photo Analysis JSON**
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Executes JavaScript to parse the Gemini response. Normalizes placeholder dates (`+1234567890`, `0`, `"0"`) to `null`.
    - *Connections:* Input: `Analyze Receipt Photo with Gemini`; Output: `Check Photo Analysis Confidence`.
  - **Parse Document Analysis JSON**
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Identical logic to the photo JSON parser, sanitizing dates and parsing the JSON payload.
    - *Connections:* Input: `Analyze Receipt Document with Gemini`; Output: `Check Document Analysis Confidence`.
  - **Check Photo Analysis Confidence**
    - *Type/Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Compares confidence score against the threshold.
    - *Expressions:* `={{ $json.confidence }}` > `0.65`.
    - *Connections:* Input: `Parse Photo Analysis JSON`; Output: True branch to `Append Photo Expense to Sheets`, False branch to `Prepare Expense for Review`.
  - **Check Document Analysis Confidence**
    - *Type/Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Compares confidence score against the threshold.
    - *Expressions:* `={{ $json.confidence }}` > `0.65`.
    - *Connections:* Input: `Parse Document Analysis JSON`; Output: True branch to `Append Document Expense to Sheets`, False branch to `Prepare Expense for Review`.

---

#### 2.4 Human-in-the-Loop Review & Callback Handling
- **Overview:** Temporarily stages low-confidence expenses in an n8n Data Table, pauses workflow execution using a Wait node, prompts the user in Telegram via inline buttons, and processes incoming callback queries to resume execution with confirmed or modified categories.
- **Nodes Involved:**
  - `Prepare Expense for Review`
  - `Save Pending Expense to Data Table`
  - `Confirm Expense Category on Telegram`
  - `Wait for Expense Category Approval`
  - `Parse Telegram Callback Data`
  - `Answer Telegram Callback Query`
  - `Route by Category Response`
  - `Get Expense for Approval from Table`
  - `Post to Resume Category Workflow`
  - `Prompt Category Selection on Telegram`
  - `Get Expense for New Category from Table`
  - `Post to Resume New Category Workflow`
- **Node Details:**
  - **Prepare Expense for Review**
    - *Type/Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Assigns execution metadata (`request_id` as `$execution.id`, `resume_url` as `$execution.resumeUrl`) alongside expense attributes.
    - *Connections:* Input: `Check Photo Analysis Confidence` / `Check Document Analysis Confidence` (False branches); Output: `Save Pending Expense to Data Table`.
  - **Save Pending Expense to Data Table**
    - *Type/Role:* `n8n-nodes-base.dataTable` (Database Operation)
    - *Configuration:* Operation: `append`. Maps fields to the `pending_expenses` data table (`request_id`, `resume_url`, `vendor`, `amount`, `date`, `category`, `payment_method`).
    - *Connections:* Input: `Prepare Expense for Review`; Output: `Confirm Expense Category on Telegram`.
  - **Confirm Expense Category on Telegram**
    - *Type/Role:* `n8n-nodes-base.telegram` (Action / Message)
    - *Configuration:* Sends a message asking to confirm the category, attaching an inline keyboard with "Yes" and "No" buttons containing callback data (`=yes_{{ $execution.id }}` and `=no_{{ $execution.id }}`).
    - *Connections:* Input: `Save Pending Expense to Data Table`; Output: `Wait for Expense Category Approval`.
  - **Wait for Expense Category Approval**
    - *Type/Role:* `n8n-nodes-base.wait` (Flow Control / Pause)
    - *Configuration:* Resume method: `webhook`, HTTP Method: `POST`. Pauses the workflow execution until an external HTTP request hits the resume URL.
    - *Connections:* Input: `Confirm Expense Category on Telegram`; Output: `Append Reviewed Expense to Sheets`.
  - **Parse Telegram Callback Data**
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Splits incoming callback strings by underscores to extract the action type (`yes`, `no`, or `category`), optional values, request ID, and chat ID.
    - *Connections:* Input: `Route by Telegram Input` (Branch 3); Output: `Route by Category Response` and `Answer Telegram Callback Query`.
  - **Answer Telegram Callback Query**
    - *Type/Role:* `n8n-nodes-base.telegram` (Action / Callback)
    - *Configuration:* Resource: `callback`. Acknowledges the button press to clear loading states in the Telegram client.
    - *Expressions:* Query ID: `={{ $('Telegram Message Trigger').item.json.callback_query.id }}`.
    - *Connections:* Input: `Parse Telegram Callback Data`; Output: None.
  - **Route by Category Response**
    - *Type/Role:* `n8n-nodes-base.switch` (Flow Control)
    - *Configuration:* Evaluates the parsed `action` field (`yes`, `no`, or `category`).
    - *Connections:* Input: `Parse Telegram Callback Data`; Output: Branch 0 to `Get Expense for Approval from Table`, Branch 1 to `Prompt Category Selection on Telegram`, Branch 2 to `Get Expense for New Category from Table`.
  - **Get Expense for Approval from Table**
    - *Type/Role:* `n8n-nodes-base.dataTable` (Database Operation)
    - *Configuration:* Operation: `get`. Filters records by `request_id`.
    - *Connections:* Input: `Route by Category Response` (Branch 0); Output: `Post to Resume Category Workflow`.
  - **Post to Resume Category Workflow**
    - *Type/Role:* `n8n-nodes-base.httpRequest` (Action / HTTP Request)
    - *Configuration:* Method: `POST`. Sends an HTTP request to the pending execution's `resume_url`.
    - *Connections:* Input: `Get Expense for Approval from Table`; Output: None (Resumes paused workflow execution).
  - **Prompt Category Selection on Telegram**
    - *Type/Role:* `n8n-nodes-base.telegram` (Action / Message)
    - *Configuration:* Sends a prompt with an inline keyboard offering alternative categories (Food, Transport, Shopping, Services, Other).
    - *Connections:* Input: `Route by Category Response` (Branch 1); Output: None.
  - **Get Expense for New Category from Table**
    - *Type/Role:* `n8n-nodes-base.dataTable` (Database Operation)
    - *Configuration:* Operation: `get`. Retrieves pending record using `request_id` from the callback payload.
    - *Connections:* Input: `Route by Category Response` (Branch 2); Output: `Post to Resume New Category Workflow`.
  - **Post to Resume New Category Workflow**
    - *Type/Role:* `n8n-nodes-base.httpRequest` (Action / HTTP Request)
    - *Configuration:* Method: `POST`, Body Content Type: JSON. Posts a payload confirming review and supplying the newly selected category (`confirmed: true`, `category`).
    - *Connections:* Input: `Get Expense for New Category from Table`; Output: None (Resumes paused workflow execution with updated data).

---

#### 2.5 Data Persistence & Confirmation
- **Overview:** Writes validated or reviewed expense records into Google Sheets, removes temporary entries from the Data Table, and confirms success to the user via Telegram.
- **Nodes Involved:**
  - `Append Photo Expense to Sheets`
  - `Append Document Expense to Sheets`
  - `Append Reviewed Expense to Sheets`
  - `Delete Pending Expense from Table`
  - `Confirm Expense Saved on Telegram`
- **Node Details:**
  - **Append Photo Expense to Sheets**
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Action / Spreadsheet Append)
    - *Configuration:* Operation: `append`. Maps fields to target Document ID and Sheet ID.
    - *Credentials:* Google Sheets OAuth2 API.
    - *Connections:* Input: `Check Photo Analysis Confidence` (True branch); Output: `Confirm Expense Saved on Telegram`.
  - **Append Document Expense to Sheets**
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Action / Spreadsheet Append)
    - *Configuration:* Operation: `append`. Maps fields to target Document ID and Sheet ID.
    - *Credentials:* Google Sheets OAuth2 API.
    - *Connections:* Input: `Check Document Analysis Confidence` (True branch); Output: `Confirm Expense Saved on Telegram`.
  - **Append Reviewed Expense to Sheets**
    - *Type/Role:* `n8n-nodes-base.googleSheets` (Action / Spreadsheet Append)
    - *Configuration:* Operation: `append`. Appends the reviewed/updated expense data to the target sheet.
    - *Credentials:* Google Sheets OAuth2 API.
    - *Connections:* Input: `Wait for Expense Category Approval`; Output: `Delete Pending Expense from Table`.
  - **Delete Pending Expense from Table**
    - *Type/Role:* `n8n-nodes-base.dataTable` (Database Operation)
    - *Configuration:* Operation: `deleteRows`. Removes the temporary entry from the Data Table using the record ID.
    - *Connections:* Input: `Append Reviewed Expense to Sheets`; Output: `Confirm Expense Saved on Telegram`.
  - **Confirm Expense Saved on Telegram**
    - *Type/Role:* `n8n-nodes-base.telegram` (Action / Message)
    - *Configuration:* Sends success confirmation text ("Expense saved successfully.") to the user.
    - *Connections:* Input: `Append Photo Expense to Sheets`, `Append Document Expense to Sheets`, or `Delete Pending Expense from Table`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram Message Trigger | `n8n-nodes-base.telegramTrigger` | Triggers workflow on incoming Telegram messages or callbacks. | None | Route by Telegram Input | ## AI Receipt Processor<br><br>### How it works<br><br>This workflow lets users send receipt photos or documents to a Telegram bot. Gemini extracts the vendor, amount, date, expense category, payment method, and confidence score.<br><br>High-confidence expenses are saved directly to Google Sheets. Low-confidence expenses are stored temporarily in an n8n Data Table and the user is asked to confirm or change the category in Telegram.<br><br>### Requirements<br><br>- Telegram bot credentials<br>- Google Gemini credentials<br>- Google Sheets credentials<br>- an n8n Data Table named `pending_expenses`<br>- a Google Sheet for storing expenses<br><br>### Setup steps<br><br>1. Connect Telegram credentials to the trigger and all Telegram nodes.<br>2. Connect Google Gemini credentials to both Gemini nodes and select a Gemini model.<br>3. Make a copy of the provided **Expense template** and select it in all three Google Sheets nodes - https://docs.google.com/spreadsheets/d/129ZEubOfaVA1pOjbVCb9GgZQXbC48XTGpH7Zve5hBU0/edit?usp=sharing<br><br>4. Map these Google Sheets columns:<br>   - `Date`<br>   - `Vendor`<br>   - `Amount`<br>   - `Payment type`<br>   - `Expense Category `<br>5. Create an n8n Data Table named `pending_expenses` with:<br>   - `request_id` — Number<br>   - `resume_url` — String<br>   - `vendor` — String<br>   - `amount` — Number<br>   - `date` — Date/Time<br>   - `category` — String<br>   - `payment_method` — String<br>6. Select this Data Table in all Data Table nodes.<br>7. If you are self-hosting n8n, ensure your instance is publicly reachable for Telegram updates and wait/resume URLs.<br>8. Activate the workflow and send a receipt photo or document to your Telegram bot.<br><br>### Customization<br><br>You can customize the confidence threshold, Gemini extraction prompt, expense categories, Telegram messages, Google Sheets columns, and category confirmation buttons.<br><br>The default confidence threshold is `0.65`.<br><br>The default categories are `Food`, `Transport`, `Shopping`, `Services`, and `Other`. |
| Route by Telegram Input | `n8n-nodes-base.switch` | Routes Telegram payloads based on content type (photo, document, text, callback). | Telegram Message Trigger | Download Telegram Photo, Download Telegram Document, Handle Unsupported Text Input, Parse Telegram Callback Data | ## Receive and route input<br><br>Accepts Telegram messages or callback queries and branches them into photo, document, unsupported text, or callback-processing paths. |
| Download Telegram Photo | `n8n-nodes-base.telegram` | Downloads the binary photo file from Telegram servers. | Route by Telegram Input | Analyze Receipt Photo with Gemini | ## Analyze photo receipt<br><br>Downloads an uploaded Telegram photo and sends it to Gemini for receipt extraction. |
| Download Telegram Document | `n8n-nodes-base.telegram` | Downloads the binary document file from Telegram servers. | Route by Telegram Input | Analyze Receipt Document with Gemini | ## Analyze document receipt<br><br>Downloads an uploaded Telegram document and sends it to Gemini for receipt extraction. |
| Handle Unsupported Text Input | `n8n-nodes-base.telegram` | Replies to the user when non-file text is sent. | Route by Telegram Input | None | ## Handle unsupported text<br><br>Sends a Telegram response when the incoming message is plain text instead of a receipt photo or document. |
| Analyze Receipt Photo with Gemini | `@n8n/n8n-nodes-langchain.googleGemini` | Extracts structured expense details from receipt images using Gemini. | Download Telegram Photo | Parse Photo Analysis JSON | ## Analyze photo receipt<br><br>Downloads an uploaded Telegram photo and sends it to Gemini for receipt extraction. |
| Analyze Receipt Document with Gemini | `@n8n/n8n-nodes-langchain.googleGemini` | Extracts structured expense details from receipt documents using Gemini. | Download Telegram Document | Parse Document Analysis JSON | ## Analyze document receipt<br><br>Downloads an uploaded Telegram document and sends it to Gemini for receipt extraction. |
| Parse Photo Analysis JSON | `n8n-nodes-base.code` | Parses Gemini's photo analysis output and normalizes dates. | Analyze Receipt Photo with Gemini | Check Photo Analysis Confidence | ## Validate photo extraction<br><br>Parses Gemini's photo analysis JSON and checks whether the extracted receipt data is confident enough to save automatically. |
| Parse Document Analysis JSON | `n8n-nodes-base.code` | Parses Gemini's document analysis output and normalizes dates. | Analyze Receipt Document with Gemini | Check Document Analysis Confidence | ## Validate document extraction<br><br>Parses Gemini's document analysis JSON and checks whether the extracted receipt data is confident enough to save automatically. |
| Check Photo Analysis Confidence | `n8n-nodes-base.if` | Checks if photo extraction confidence exceeds 0.65. | Parse Photo Analysis JSON | Append Photo Expense to Sheets, Prepare Expense for Review | ## Validate photo extraction<br><br>Parses Gemini's photo analysis JSON and checks whether the extracted receipt data is confident enough to save automatically. |
| Check Document Analysis Confidence | `n8n-nodes-base.if` | Checks if document extraction confidence exceeds 0.65. | Parse Document Analysis JSON | Append Document Expense to Sheets, Prepare Expense for Review | ## Validate document extraction<br><br>Parses Gemini's document analysis JSON and checks whether the extracted receipt data is confident enough to save automatically. |
| Append Photo Expense to Sheets | `n8n-nodes-base.googleSheets` | Appends high-confidence photo expense data to Google Sheets. | Check Photo Analysis Confidence | Confirm Expense Saved on Telegram | ## Save and notify expense<br><br>Writes automatic photo/document expenses or reviewed expenses to Google Sheets, removes reviewed pending records, and sends the final Telegram saved confirmation. |
| Append Document Expense to Sheets | `n8n-nodes-base.googleSheets` | Appends high-confidence document expense data to Google Sheets. | Check Document Analysis Confidence | Confirm Expense Saved on Telegram | ## Save and notify expense<br><br>Writes automatic photo/document expenses or reviewed expenses to Google Sheets, removes reviewed pending records, and sends the final Telegram saved confirmation. |
| Prepare Expense for Review | `n8n-nodes-base.set` | Prepares low-confidence expense data with execution IDs and resume URLs. | Check Photo Analysis Confidence, Check Document Analysis Confidence | Save Pending Expense to Data Table | ## Store pending expense<br><br>Prepares the low-confidence expense data with request and resume information, then stores it in the Data Table for later review. |
| Save Pending Expense to Data Table | `n8n-nodes-base.dataTable` | Stores low-confidence expense details into the Data Table. | Prepare Expense for Review | Confirm Expense Category on Telegram | ## Store pending expense<br><br>Prepares the low-confidence expense data with request and resume information, then stores it in the Data Table for later review. |
| Confirm Expense Category on Telegram | `n8n-nodes-base.telegram` | Asks the user to confirm the suggested category via inline buttons. | Save Pending Expense to Data Table | Wait for Expense Category Approval | ## Request category approval<br><br>Asks the user in Telegram to confirm the suggested category and pauses the workflow until a callback resumes it. |
| Wait for Expense Category Approval | `n8n-nodes-base.wait` | Pauses workflow execution until a resume webhook is received. | Confirm Expense Category on Telegram | Append Reviewed Expense to Sheets | ## Request category approval<br><br>Asks the user in Telegram to confirm the suggested category and pauses the workflow until a callback resumes it. |
| Parse Telegram Callback Data | `n8n-nodes-base.code` | Parses inline keyboard callback actions and request IDs. | Route by Telegram Input | Route by Category Response, Answer Telegram Callback Query | ## Parse callback response<br><br>Processes incoming Telegram callback payloads and immediately answers the callback query so Telegram acknowledges the button press. |
| Answer Telegram Callback Query | `n8n-nodes-base.telegram` | Acknowledges the Telegram callback query to clear loading states. | Parse Telegram Callback Data | None | ## Parse callback response<br><br>Processes incoming Telegram callback payloads and immediately answers the callback query so Telegram acknowledges the button press. |
| Route by Category Response | `n8n-nodes-base.switch` | Routes callback action types (`yes`, `no`, `category`). | Parse Telegram Callback Data | Get Expense for Approval from Table, Prompt Category Selection on Telegram, Get Expense for New Category from Table | ## Route category choices<br><br>Routes the parsed callback into confirm, choose-category, or new-category flows, and can prompt the user to pick a different category. |
| Get Expense for Approval from Table | `n8n-nodes-base.dataTable` | Retrieves the pending expense record for confirmation. | Route by Category Response | Post to Resume Category Workflow | ## Resume confirmed approval<br><br>Fetches the pending expense for a confirmed category and posts to its resume URL to continue the paused workflow. |
| Post to Resume Category Workflow | `n8n-nodes-base.httpRequest` | Triggers the resume URL to continue the paused execution. | Get Expense for Approval from Table | None | ## Resume confirmed approval<br><br>Fetches the pending expense for a confirmed category and posts to its resume URL to continue the paused workflow. |
| Prompt Category Selection on Telegram | `n8n-nodes-base.telegram` | Prompts the user to select an alternative expense category. | Route by Category Response | None | ## Route category choices<br><br>Routes the parsed callback into confirm, choose-category, or new-category flows, and can prompt the user to pick a different category. |
| Get Expense for New Category from Table | `n8n-nodes-base.dataTable` | Retrieves the pending expense record for category modification. | Route by Category Response | Post to Resume New Category Workflow | ## Resume new category<br><br>Fetches the pending expense associated with a newly selected category and posts to its resume URL to continue processing. |
| Post to Resume New Category Workflow | `n8n-nodes-base.httpRequest` | Posts updated category data to the resume URL to continue execution. | Get Expense for New Category from Table | None | ## Resume new category<br><br>Fetches the pending expense associated with a newly selected category and posts to its resume URL to continue processing. |
| Append Reviewed Expense to Sheets | `n8n-nodes-base.googleSheets` | Appends reviewed and approved expense data to Google Sheets. | Wait for Expense Category Approval | Delete Pending Expense from Table | ## Save and notify expense<br><br>Writes automatic photo/document expenses or reviewed expenses to Google Sheets, removes reviewed pending records, and sends the final Telegram saved confirmation. |
| Delete Pending Expense from Table | `n8n-nodes-base.dataTable` | Cleans up the temporary record from the Data Table after approval. | Append Reviewed Expense to Sheets | Confirm Expense Saved on Telegram | ## Save and notify expense<br><br>Writes automatic photo/document expenses or reviewed expenses to Google Sheets, removes reviewed pending records, and sends the final Telegram saved confirmation. |
| Confirm Expense Saved on Telegram | `n8n-nodes-base.telegram` | Sends success notification message to the Telegram user. | Append Photo Expense to Sheets, Append Document Expense to Sheets, Delete Pending Expense from Table | None | ## Save and notify expense<br><br>Writes automatic photo/document expenses or reviewed expenses to Google Sheets, removes reviewed pending records, and sends the final Telegram saved confirmation. |

---

### 4. Reproducing the Workflow from Scratch

#### Prerequisites & Setup
1. **Credentials Required:**
   - **Telegram API:** Connected bot credential.
   - **Google Gemini (PaLM) API:** Active API account credential.
   - **Google Sheets OAuth2 API:** Connected account with access to your target spreadsheet.
2. **External Data Structures:**
   - **Google Sheet:** Create a spreadsheet containing columns: `Date`, `Vendor`, `Amount`, `Payment type`, and `Expense Category`.
   - **n8n Data Table:** Create a Data Table named `pending_expenses` with columns: `request_id` (Number), `resume_url` (String), `vendor` (String), `amount` (Number), `date` (Date/Time), `category` (String), and `payment_method` (String).

#### Step-by-Step Build Instructions
1. **Create the Trigger:**
   - Add a `Telegram Message Trigger` node. Set updates to `message` and `callback_query`. Link your Telegram API credential.
2. **Add Routing Switch:**
   - Create a `Switch` node (`Route by Telegram Input`) connected to the trigger. Configure 4 rules checking for `message.photo`, `message.document`, `message.text`, and `callback_query`.
3. **Build the Photo Extraction Branch:**
   - Add a `Telegram` node (`Download Telegram Photo`) set to resource `file` with expression `={{ $json.message.photo[$json.message.photo.length - 1].file_id }}`.
   - Add a Google Gemini node (`Analyze Receipt Photo with Gemini`) configured for image analysis using binary input. Set retry on fail.
   - Add a `Code` node (`Parse Photo Analysis JSON`) to parse the JSON output and normalize placeholder dates to `null`.
   - Add an `If` node (`Check Photo Analysis Confidence`) checking if `={{ $json.confidence }}` > `0.65`.
4. **Build the Document Extraction Branch:**
   - Add a `Telegram` node (`Download Telegram Document`) set to resource `file` with expression `={{ $json.message.document.file_id }}`.
   - Add a Google Gemini node (`Analyze Receipt Document with Gemini`) configured for document analysis using binary input.
   - Add a `Code` node (`Parse Document Analysis JSON`) identical in logic to the photo parser.
   - Add an `If` node (`Check Document Analysis Confidence`) checking if `={{ $json.confidence }}` > `0.65`.
5. **Handle Unsupported Text:**
   - Add a `Telegram` node (`Handle Unsupported Text Input`) to reply with an error message when plain text is received.
6. **Build the Automated Save Branches:**
   - Add two `Google Sheets` nodes (`Append Photo Expense to Sheets` and `Append Document Expense to Sheets`), set to append operation, mapping the extracted fields to your spreadsheet columns.
   - Connect the True outputs of both confidence `If` nodes to these respective append nodes.
7. **Build the Low-Confidence Review Branch:**
   - Add a `Set` node (`Prepare Expense for Review`) mapping execution metadata (`request_id` = `={{ $execution.id }}`, `resume_url` = `={{ $execution.resumeUrl }}`) and receipt data.
   - Add a `Data Table` node (`Save Pending Expense to Data Table`) configured to append to `pending_expenses`.
   - Add a `Telegram` node (`Confirm Expense Category on Telegram`) sending an inline keyboard with "Yes" and "No" buttons containing callbacks (`=yes_{{ $execution.id }}` and `=no_{{ $execution.id }}`).
   - Add a `Wait` node (`Wait for Expense Category Approval`) configured to resume via webhook (`POST`).
8. **Build the Callback and Review Resolution Branches:**
   - Add a `Code` node (`Parse Telegram Callback Data`) to extract action types and request IDs from incoming callback queries.
   - Add a `Telegram` node (`Answer Telegram Callback Query`) to acknowledge the callback query.
   - Add a `Switch` node (`Route by Category Response`) to branch based on `action` (`yes`, `no`, or `category`).
   - For confirmed categories (`yes`), add a `Data Table` node (`Get Expense for Approval from Table`) to fetch the record, followed by an `HTTP Request` node (`Post to Resume Category Workflow`) targeting `resume_url`.
   - For category rejection (`no`), add a `Telegram` node (`Prompt Category Selection on Telegram`) displaying category selection buttons.
   - For new category selections (`category`), add a `Data Table` node (`Get Expense for New Category from Table`) and an `HTTP Request` node (`Post to Resume New Category Workflow`) posting `{ "confirmed": true, "category": "..." }` to the resume URL.
9. **Finalize Persistence & Confirmation:**
   - Connect the output of the `Wait` node to a `Google Sheets` node (`Append Reviewed Expense to Sheets`) to log approved expenses.
   - Connect that append node to a `Data Table` node (`Delete Pending Expense from Table`) to remove the temporary entry.
   - Add a final `Telegram` node (`Confirm Expense Saved on Telegram`) to send success messages for all completion paths.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Expense template spreadsheet | [Google Sheets Template](https://docs.google.com/spreadsheets/d/129ZEubOfaVA1pOjbVCb9GgZQXbC48XTGpH7Zve5hBU0/edit?usp=sharing) |