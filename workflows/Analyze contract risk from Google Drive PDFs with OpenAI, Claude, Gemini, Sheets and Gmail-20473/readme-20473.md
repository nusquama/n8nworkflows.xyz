Analyze contract risk from Google Drive PDFs with OpenAI, Claude, Gemini, Sheets and Gmail

https://n8nworkflows.xyz/workflows/analyze-contract-risk-from-google-drive-pdfs-with-openai--claude--gemini--sheets-and-gmail-20473


# Analyze contract risk from Google Drive PDFs with OpenAI, Claude, Gemini, Sheets and Gmail

### 1. Workflow Overview

This workflow is designed to automate the initial intake, evaluation, and risk assessment of incoming PDF contracts stored within Google Drive. It is primarily used by legal, operations, or procurement teams to accelerate contract reviews, maintain a centralized audit log, and instantly flag high-risk agreements.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Watches a specified Google Drive folder for newly uploaded files, filters for PDFs, and initializes execution parameters (such as the target AI provider, Google Sheet details, risk thresholds, and character limits).
- **1.2 Document Retrieval & Preparation:** Downloads the target PDF from Google Drive, extracts its raw text, normalizes whitespace, and truncates the content to fit within a safe character limit for LLM processing.
- **1.3 Multi-Model AI Routing & Analysis:** Evaluates the chosen AI provider configuration and routes the prepared contract text to either OpenAI (GPT-4o-mini), Anthropic (Claude Sonnet), or Google Gemini. Each model uses a structured information extractor to pull key metadata, financial terms, risk flags, and an overall risk score (0–100).
- **1.4 Normalization, Logging & Notification:** Standardizes the output from whichever AI model was executed, appends a flat record to a Google Sheets document, evaluates whether the risk score breaches the configured alert threshold, and dispatches a detailed notification via Gmail if necessary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** This block monitors a specific Google Drive directory for new additions, ensures that only PDF files proceed through the workflow, and injects global configuration variables that control downstream behavior.
- **Nodes Involved:** 
  - `When Contract Added to Drive`
  - `Filter PDF Contracts`
  - `Set Processing Parameters`
- **Node Details:**
  - **When Contract Added to Drive**
    - *Type & Technical Role:* `n8n-nodes-base.googleDriveTrigger` (Trigger Node). Polls a specific Google Drive folder every minute for newly created files.
    - *Configuration Choices:* Watches a specific folder ID (`folderToWatch`) using a polling frequency of 1 minute (`pollTimes`).
    - *Key Expressions:* None (uses native trigger parameters).
    - *Input/Output:* Input: None; Output: File metadata object (`$json.id`, `$json.name`, `$json.mimeType`, `$json.webViewLink`).
    - *Edge Cases / Failures:* API rate limits or invalid Google Drive credentials will disrupt polling. Scanned non-searchable PDFs without OCR will pass the filter but fail text extraction later.
  - **Filter PDF Contracts**
    - *Type & Technical Role:* `n8n-nodes-base.filter` (Flow Control Node). Drops execution flow for any file that is not a PDF.
    - *Configuration Choices:* Evaluates conditions using loose type validation with an AND combinator.
    - *Key Expressions:* `={{ $json.mimeType }}` must equal `application/pdf`.
    - *Input/Output:* Input: `When Contract Added to Drive`; Output: Filtered array containing only PDF items.
    - *Edge Cases / Failures:* Files with missing or incorrect MIME types will be incorrectly filtered out.
  - **Set Processing Parameters**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation Node). Assigns configuration variables used by downstream nodes.
    - *Configuration Choices:* Sets static string and numeric assignments: `aiProvider` (`openai`), `googleSheetId` (`YOUR_GOOGLE_SHEET_ID`), `sheetTabName` (`Contracts`), `alertEmail` (`user@example.com`), `riskAlertThreshold` (`70`), and `maxCharacters` (`60000`).
    - *Key Expressions:* None (static values).
    - *Input/Output:* Input: `Filter PDF Contracts`; Output: File metadata combined with configuration fields.
    - *Edge Cases / Failures:* Placeholder IDs (`YOUR_GOOGLE_SHEET_ID`) must be updated with valid target resource IDs before execution.

#### 2.2 Document Retrieval & Preparation
- **Overview:** This block downloads the physical file from Google Drive, extracts the internal text layer from the PDF, cleans formatting irregularities, and trims the content down to prevent token overflow.
- **Nodes Involved:**
  - `Download PDF from Drive`
  - `Extract PDF Text`
  - `Prepare Text for Analysis`
- **Node Details:**
  - **Download PDF from Drive**
    - *Type & Technical Role:* `n8n-nodes-base.googleDrive` (Action Node). Downloads binary file content from Google Drive.
    - *Configuration Choices:* Operation set to `download` utilizing the file ID from the upstream trigger.
    - *Key Expressions:* `={{ $json.id }}`
    - *Input/Output:* Input: `Set Processing Parameters`; Output: Binary file object containing the PDF data.
    - *Edge Cases / Failures:* File permission errors or deleted files will throw an execution error.
  - **Extract PDF Text**
    - *Type & Technical Role:* `n8n-nodes-base.extractFromFile` (Data Extraction Node). Extracts readable text strings from binary PDF data.
    - *Configuration Choices:* Operation set to `pdf`.
    - *Input/Output:* Input: `Download PDF from Drive`; Output: Text payload (`$json.text`).
    - *Edge Cases / Failures:* Password-protected PDFs or scanned images without embedded text layers will result in empty text strings, triggering validation errors in the subsequent code node.
  - **Prepare Text for Analysis**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Custom JavaScript Node). Cleans whitespace, checks for empty text content, and slices the text to the configured maximum character limit.
    - *Configuration Choices:* Executed in `runOnceForEachItem` mode.
    - *Key Expressions:* References `$('Set Processing Parameters').item.json` to fetch `maxCharacters` and file metadata.
    - *Input/Output:* Input: `Extract PDF Text`; Output: Normalized JSON payload containing `contractText`, `truncated`, `characterCount`, and metadata fields.
    - *Edge Cases / Failures:* Throws an explicit error if `contractText` is empty, indicating that the PDF requires OCR processing.

#### 2.3 Multi-Model AI Routing & Analysis
- **Overview:** This block dynamically routes the prepared contract text to one of three AI analysis branches (OpenAI, Anthropic Claude, or Google Gemini) based on the user's `aiProvider` configuration. Each branch uses a structured information extractor to map the contract text to a predefined schema.
- **Nodes Involved:**
  - `Route by AI Provider`
  - `OpenAI Contract Analysis`
  - `OpenAI GPT-4 Model`
  - `Claude Contract Analysis`
  - `Claude Sonnet Model`
  - `Gemini Contract Analysis`
  - `Gemini Chat Model`
- **Node Details:**
  - **Route by AI Provider**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Flow Control Node). Directs execution to output index 0 (OpenAI), 1 (Anthropic), or 2 (Gemini).
    - *Configuration Choices:* Expression mode routing with 3 outputs.
    - *Key Expressions:* `={{ { openai: 0, anthropic: 1, gemini: 2 }[$json.aiProvider] ?? 0 }}`
    - *Input/Output:* Input: `Prepare Text for Analysis`; Output: Routes execution to the matching AI provider branch.
    - *Edge Cases / Failures:* Unrecognized provider strings default to index 0 (OpenAI).
  - **OpenAI Contract Analysis**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.informationExtractor` (AI / LangChain Node). Extracts structured JSON metadata from text using a specified schema.
    - *Configuration Choices:* Uses a manual schema defining contract titles, types, parties, dates, financial values, risk flags (with severities), risk scores, and summaries. System prompt instructs the model to act as an experienced contract analyst.
    - *Key Expressions:* `=Contract file: {{ $json.fileName }}\n\n{{ $json.contractText }}`
    - *Input/Output:* Input: `Route by AI Provider` (Index 0); Output: Structured AI extraction object (`$json.output`).
    - *Edge Cases / Failures:* API rate limits, invalid OpenAI credentials, or model timeouts.
  - **OpenAI GPT-4 Model**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (LangChain Sub-Node). Provides the underlying language model for the OpenAI extractor.
    - *Configuration Choices:* Model set to `gpt-4o-mini` with temperature configured to `0.1`.
    - *Input/Output:* Connected via AI language model wire to `OpenAI Contract Analysis`.
  - **Claude Contract Analysis**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.informationExtractor` (AI / LangChain Node). Identical extraction schema and system prompt configured to run on Anthropic infrastructure.
    - *Key Expressions:* `=Contract file: {{ $json.fileName }}\n\n{{ $json.contractText }}`
    - *Input/Output:* Input: `Route by AI Provider` (Index 1); Output: Structured AI extraction object (`$json.output`).
    - *Edge Cases / Failures:* Anthropic API limits or invalid credentials.
  - **Claude Sonnet Model**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatAnthropic` (LangChain Sub-Node). Provides the language model (`claude-sonnet-4-5`) with a temperature of `0.1`.
    - *Input/Output:* Connected via AI language model wire to `Claude Contract Analysis`.
  - **Gemini Contract Analysis**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.informationExtractor` (AI / LangChain Node). Extracts contract metadata using Google's Gemini models.
    - *Key Expressions:* `=Contract file: {{ $json.fileName }}\n\n{{ $json.contractText }}`
    - *Input/Output:* Input: `Route by AI Provider` (Index 2); Output: Structured AI extraction object (`$json.output`).
    - *Edge Cases / Failures:* Google API limits or invalid credentials.
  - **Gemini Chat Model**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (LangChain Sub-Node). Configures Google Gemini chat generation settings (temperature `0.1`).
    - *Input/Output:* Connected via AI language model wire to `Gemini Contract Analysis`.

#### 2.4 Normalization, Logging & Notification
- **Overview:** This block standardizes the outputs from whichever AI model ran, logs the structured analysis as a new row in Google Sheets, checks whether the calculated risk score breaches the alert threshold, and sends a styled HTML email via Gmail if required.
- **Nodes Involved:**
  - `Normalize AI Analysis`
  - `Append Analysis to Sheets`
  - `Check Risk Level`
  - `Send Risk Alert via Email`
- **Node Details:**
  - **Normalize AI Analysis**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Custom JavaScript Node). Flattens and formats arrays, objects, and scoring metrics into a consistent schema suitable for spreadsheet insertion and email notifications.
    - *Configuration Choices:* Executed in `runOnceForEachItem` mode. Calculates risk levels (`High`, `Medium`, `Low`) based on score thresholds (High >= 70, Medium >= 40) and determines alert flags.
    - *Key Expressions:* References upstream item data via `$('Prepare Text for Analysis').item.json`, `$('Set Processing Parameters').item.json`, and `$json.output`.
    - *Input/Output:* Input: Any of the three AI Information Extractor nodes; Output: Flat, normalized JSON object.
    - *Edge Cases / Failures:* Malformed JSON structures from the LLM are safely handled with fallback logical operators (e.g., `Number(a.riskScore) || 0`).
  - **Append Analysis to Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action Node). Appends a new row of data to a Google Sheets document.
    - *Configuration Choices:* Operation set to `append`. Uses auto-mapping for input data mapping columns.
    - *KeyExpressions:* 
      - Spreadsheet ID: `={{ $('Set Processing Parameters').item.json.googleSheetId }}`
      - Sheet Name: `={{ $('Set Processing Parameters').item.json.sheetTabName }}`
    - *Input/Output:* Input: `Normalize AI Analysis`; Output: Confirmation object containing updated sheet row details.
    - *Edge Cases / Failures:* Sheet permission errors, missing columns, or mismatched sheet tab names.
  - **Check Risk Level**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control Node). Evaluates whether an email alert should be sent based on the normalized alert decision.
    - *Configuration Choices:* Checks boolean truthiness of the alert flag.
    - *Key Expressions:* `={{ $('Normalize AI Analysis').item.json.sendAlert }}`
    - *Input/Output:* Input: `Append Analysis to Sheets`; Output: True branch (proceeds to email alert), False branch (terminates execution).
    - *Edge Cases / Failures:* None if upstream normalization properties evaluate correctly.
  - **Send Risk Alert via Email**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Action Node). Sends an HTML-formatted email alert via Gmail.
    - *Configuration Choices:* Configured with dynamic subject lines and HTML message bodies detailing risk scores, summaries, risk flags, and direct links to the source document in Google Drive.
    - *Key Expressions:*
      - Recipient: `={{ $('Set Processing Parameters').item.json.alertEmail }}`
      - Subject: `=Contract risk alert: {{ $('Normalize AI Analysis').item.json.contractTitle }} ({{ $('Normalize AI Analysis').item.json.riskScore }}/100)`
      - Message: HTML string mapping normalized fields from `Normalize AI Analysis`.
    - *Input/Output:* Input: `Check Risk Level` (True Branch); Output: Gmail API response confirming message dispatch.
    - *Edge Cases / Failures:* Invalid recipient address formats, OAuth scope restrictions, or Gmail daily sending limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Analyze contracts from Google Drive with AI... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Watch and configure input... |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Download and prepare PDF... |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Route AI provider... |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## OpenAI contract analysis... |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Claude contract analysis... |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Gemini contract analysis... |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Normalize log and alert... |
| When Contract Added to Drive | `n8n-nodes-base.googleDriveTrigger` | Trigger workflow on new files | None | Filter PDF Contracts | ## Watch and configure input... |
| Filter PDF Contracts | `n8n-nodes-base.filter` | Filter out non-PDF files | When Contract Added to Drive | Set Processing Parameters | ## Watch and configure input... |
| Set Processing Parameters | `n8n-nodes-base.set` | Define global execution variables | Filter PDF Contracts | Download PDF from Drive | ## Watch and configure input... |
| Download PDF from Drive | `n8n-nodes-base.googleDrive` | Download contract file binary | Set Processing Parameters | Extract PDF Text | ## Download and prepare PDF... |
| Extract PDF Text | `n8n-nodes-base.extractFromFile` | Extract text layer from PDF | Download PDF from Drive | Prepare Text for Analysis | ## Download and prepare PDF... |
| Prepare Text for Analysis | `n8n-nodes-base.code` | Clean and truncate text | Extract PDF Text | Route by AI Provider | ## Download and prepare PDF... |
| Route by AI Provider | `n8n-nodes-base.switch` | Route flow to selected LLM | Prepare Text for Analysis | OpenAI Contract Analysis,<br>Claude Contract Analysis,<br>Gemini Contract Analysis | ## Route AI provider... |
| OpenAI Contract Analysis | `@n8n/n8n-nodes-langchain.informationExtractor` | Extract risk data via OpenAI | Route by AI Provider | Normalize AI Analysis | ## OpenAI contract analysis |
| OpenAI GPT-4 Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM sub-node for OpenAI | None (Sub-node) | OpenAI Contract Analysis | ## OpenAI contract analysis |
| Claude Contract Analysis | `@n8n/n8n-nodes-langchain.informationExtractor` | Extract risk data via Claude | Route by AI Provider | Normalize AI Analysis | ## Claude contract analysis |
| Claude Sonnet Model | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | LLM sub-node for Claude | None (Sub-node) | Claude Contract Analysis | ## Claude contract analysis |
| Gemini Contract Analysis | `@n8n/n8n-nodes-langchain.informationExtractor` | Extract risk data via Gemini | Route by AI Provider | Normalize AI Analysis | ## Gemini contract analysis |
| Gemini Chat Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM sub-node for Gemini | None (Sub-node) | Gemini Contract Analysis | ## Gemini contract analysis |
| Normalize AI Analysis | `n8n-nodes-base.code` | Flatten and standardize AI output | OpenAI Contract Analysis,<br>Claude Contract Analysis,<br>Gemini Contract Analysis | Append Analysis to Sheets | ## Normalize log and alert |
| Append Analysis to Sheets | `n8n-nodes-base.googleSheets` | Log analysis into Google Sheets | Normalize AI Analysis | Check Risk Level | ## Normalize log and alert |
| Check Risk Level | `n8n-nodes-base.if` | Check if risk threshold is met | Append Analysis to Sheets | Send Risk Alert via Email | ## Normalize log and alert |
| Send Risk Alert via Email | `n8n-nodes-base.gmail` | Send HTML alert via Gmail | Check Risk Level | None | ## Normalize log and alert |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually:

1. **Create the Trigger Node:**
   - Add a **Google Drive Trigger** node (`n8n-nodes-base.googleDriveTrigger`).
   - Set the event to `File Created`.
   - Set the trigger mode to `Specific Folder` and select your target contracts folder. Set poll times to every minute.
   - Configure valid Google Drive OAuth2 credentials.

2. **Add the PDF Filter:**
   - Add a **Filter** node (`n8n-nodes-base.filter`) and connect it to the trigger.
   - Configure a condition where `{{ $json.mimeType }}` equals `application/pdf`.

3. **Set Processing Parameters:**
   - Add a **Set (Edit Fields)** node (`n8n-nodes-base.set`).
   - Create string assignments:
     - `aiProvider`: `openai` (or `anthropic`, `gemini`)
     - `googleSheetId`: `<Your Google Spreadsheet ID>`
     - `sheetTabName`: `Contracts`
     - `alertEmail`: `user@example.com`
   - Create numeric assignments:
     - `riskAlertThreshold`: `70`
     - `maxCharacters`: `60000`
   - Enable `Include Other Fields`.

4. **Download and Extract PDF Text:**
   - Add a **Google Drive** node (`n8n-nodes-base.googleDrive`) set to `Download` operation. Set the file ID parameter to `={{ $json.id }}`.
   - Add an **Extract From File** node (`n8n-nodes-base.extractFromFile`) set to operation `PDF`.
   - Add a **Code** node (`n8n-nodes-base.code`) named "Prepare Text for Analysis" running in `Run Once for Each Item` mode. Paste JavaScript logic to replace multiple spaces/newlines, validate text presence, and slice the text up to `maxCharacters`.

5. **Configure AI Routing:**
   - Add a **Switch** node (`n8n-nodes-base.switch`) configured for expression routing with 3 outputs:
     `={{ { openai: 0, anthropic: 1, gemini: 2 }[$json.aiProvider] ?? 0 }}`

6. **Build the OpenAI Analysis Branch:**
   - Add an **Information Extractor** node (`@n8n/n8n-nodes-langchain.informationExtractor`) connected to Switch output 0.
   - Set the input text expression to `=Contract file: {{ $json.fileName }}\n\n{{ $json.contractText }}`.
   - Define a manual schema containing properties for `contractTitle`, `contractType`, `parties`, `effectiveDate`, `expirationDate`, `autoRenewal`, `noticePeriodDays`, `totalValue`, `currency`, `paymentTerms`, `governingLaw`, `liabilityCap`, `terminationClause`, `keyObligations`, `riskFlags`, `riskScore`, `summary`, and `recommendedActions`. Set required fields to `contractTitle`, `contractType`, `parties`, `riskFlags`, `riskScore`, and `summary`.
   - Add an **OpenAI Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`), select model `gpt-4o-mini`, set temperature to `0.1`, and connect it to the Information Extractor via the AI model wire. Connect OpenAI credentials.

7. **Build the Anthropic Claude Branch:**
   - Add a second **Information Extractor** node connected to Switch output 1, duplicating the manual schema and prompt settings from the OpenAI node.
   - Add a **Claude Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatAnthropic`), select model `claude-sonnet-4-5`, set temperature to `0.1`, and connect it to the Information Extractor. Connect Anthropic credentials.

8. **Build the Google Gemini Branch:**
   - Add a third **Information Extractor** node connected to Switch output 2, duplicating the manual schema and prompt settings.
   - Add a **Google Gemini Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`), set temperature to `0.1`, and connect it to the Information Extractor. Connect Google Gemini credentials.

9. **Normalize AI Output:**
   - Add a **Code** node (`n8n-nodes-base.code`) named "Normalize AI Analysis" connected to all three Information Extractor nodes.
   - Implement JavaScript logic to parse risk scores, determine risk levels (`High`, `Medium`, `Low`), format nested arrays/objects into clean strings, and calculate the `sendAlert` boolean flag based on the `riskAlertThreshold`.

10. **Log to Google Sheets:**
    - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) set to the `Append` operation.
    - Set Document ID to `={{ $('Set Processing Parameters').item.json.googleSheetId }}` and Sheet Name to `={{ $('Set Processing Parameters').item.json.sheetTabName }}`. Use auto-mapping. Configure Google Sheets OAuth2 credentials.

11. **Check Risk Level and Send Alert:**
    - Add an **If** node (`n8n-nodes-base.if`) to check if `={{ $('Normalize AI Analysis').item.json.sendAlert }}` evaluates to `true`.
    - Connect the `true` branch to a **Gmail** node (`n8n-nodes-base.gmail`).
    - Configure the Gmail node operation to `Send`, set recipient to `={{ $('Set Processing Parameters').item.json.alertEmail }}`, set subject dynamically, and paste the HTML-formatted message body containing the contract details and file URL. Configure Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI-generated first-pass review. Not legal advice. | Standard compliance disclaimer included in prompts and email notifications. |