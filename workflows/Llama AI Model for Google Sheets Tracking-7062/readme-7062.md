Llama AI Model for Google Sheets Tracking

https://n8nworkflows.xyz/workflows/llama-ai-model-for-google-sheets-tracking-7062


# Llama AI Model for Google Sheets Tracking

### 1. Workflow Overview

This workflow automates the extraction, processing, and tracking of PDF documents linked on a specified website and integrates AI-powered summarization with Google Sheets and Discord notifications. It is designed for use cases such as monitoring company reports, financial documents, or any PDF resources published online, extracting their content, analyzing the text with an AI language model, and tracking processed data centrally in Google Sheets while notifying stakeholders via Discord.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Block**: Manual and scheduled triggers to start the workflow.
- **1.2 Website Access and PDF Link Extraction**: HTTP requests to the target website to find PDF URLs.
- **1.3 PDF Download and Text Extraction**: Downloading PDFs and extracting their textual content.
- **1.4 Google Sheets Integration**: Managing URLs and tracking processed PDFs in Google Sheets.
- **1.5 AI Processing Block**: Analyzing extracted text with an AI language model to generate summaries.
- **1.6 Discord Notification**: Sending AI-generated summaries or messages to a Discord channel.
- **1.7 Auxiliary Logic**: Data merging, filtering, and marking processed rows.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

**Overview:**  
This block initiates the workflow either manually or on a weekly schedule.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Schedule Trigger (Scheduled weekly execution)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow manually on user command.  
  - Config: Default; triggers workflow immediately when clicked.  
  - Inputs: None  
  - Outputs: Connects to HTTP Request node for website access.  
  - Edge Cases: None significant; manual activation only.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automated, weekly workflow execution.  
  - Config: Interval set to 1 week (field: weeks).  
  - Inputs: None  
  - Outputs: Connects to HTTP Request node for website access.  
  - Edge Cases: Ensure server time is accurate; missed triggers if workflow or host is offline.

---

#### 2.2 Website Access and PDF Link Extraction

**Overview:**  
This block accesses the target website, extracts PDF links, and prepares URLs for download.

**Nodes Involved:**  
- HTTP Request  
- HTML  
- Code

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Fetches the HTML content of the target website.  
  - Config: URL to the website must be set manually in parameter "url". No authentication.  
  - Inputs: Trigger nodes (manual or scheduled).  
  - Outputs: HTML node for parsing.  
  - Edge Cases: Network errors, invalid URL, or website structure changes.

- **HTML**  
  - Type: HTML Extraction  
  - Role: Extracts all `<a>` tags linking to PDFs (href ending with .pdf). Also extracts their text content (titled "tittle").  
  - Config: CSS selector `a[href$=".pdf"]`, extracts href attribute and link text. Returns arrays.  
  - Inputs: HTTP Request node output.  
  - Outputs: Code node for URL construction.  
  - Edge Cases: Website structure changes breaking selector; no PDFs found.

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Prepends base URL (`https://www.playway.com`) to relative PDF links extracted, building full URLs for download.  
  - Config: Hardcoded base URL to prepend to PDF hrefs from HTML node.  
  - Inputs: HTML node output.  
  - Outputs: Append or Update Row in Sheet node (via Google Sheets block).  
  - Edge Cases: If hrefs are absolute URLs or base URL changes, may cause wrong URLs.

---

#### 2.3 PDF Download and Text Extraction

**Overview:**  
Downloads each PDF and extracts its textual content for AI processing.

**Nodes Involved:**  
- Append or update row in sheet  
- HTTP Request1  
- Extract from File

**Node Details:**

- **Append or update row in sheet**  
  - Type: Google Sheets  
  - Role: Stores newly found PDF URLs into the 'PDFs' sheet, avoiding duplicates by matching on the "url" column.  
  - Config: Document ID and sheet name specified; mapping mode auto maps input data to columns "url" and "enviado_discord".  
  - Inputs: Code node output (list of URLs).  
  - Outputs: HTTP Request1 node for downloading PDFs.  
  - Edge Cases: Google API authentication errors; rate limiting; sheet structure must match schema.

- **HTTP Request1**  
  - Type: HTTP Request  
  - Role: Downloads each PDF via the URLs stored in Google Sheets.  
  - Config: URL parameterized from input data; response format set to "file"; never error flag to continue even on download failures; retry enabled on failure.  
  - Inputs: Append or update row in sheet output.  
  - Outputs: Extract from File node for text extraction.  
  - Edge Cases: Download failures, timeouts, file size limits.

- **Extract from File**  
  - Type: Extract from File  
  - Role: Extracts text content from the downloaded PDF file.  
  - Config: Operation set to "pdf". Continues on errors without breaking workflow.  
  - Inputs: HTTP Request1 output.  
  - Outputs: AI Agent node for text analysis and Edit Fields1 node for marking processed URLs.  
  - Edge Cases: Corrupted PDFs, non-readable files, extraction errors.

---

#### 2.4 Google Sheets Integration

**Overview:**  
Manages the storage and updating of PDF URLs and processing status in Google Sheets.

**Nodes Involved:**  
- Get row(s) in sheet  
- Merge  
- Append or update row in sheet (also part of previous block)  
- Edit Fields1  
- Update row in sheet

**Node Details:**

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Retrieves existing rows (PDF URLs) from the sheet to compare and merge new URLs.  
  - Config: Document ID and sheet "Sheet1" specified. Executes once to get current state.  
  - Inputs: Code node output (after URL construction).  
  - Outputs: Merge node for combining existing and new data.  
  - Edge Cases: API errors, sheet structure changes.

- **Merge**  
  - Type: Merge  
  - Role: Combines existing rows from Google Sheets with new URLs, matching on the "url" field to avoid duplicates.  
  - Config: Mode "combine", join mode "keepNonMatches", matching on "url".  
  - Inputs: Get row(s) in sheet and Code node.  
  - Outputs: Append or update row in sheet node.  
  - Edge Cases: Data inconsistency if URL fields differ in format.

- **Edit Fields1**  
  - Type: Set (Edit Fields)  
  - Role: Marks the current PDF URL and sets a flag "enviado_discord" to "true" to indicate the PDF has been processed and sent.  
  - Config: Sets fields "url" and "enviado_discord" = "true".  
  - Inputs: Extract from File node output.  
  - Outputs: Update row in sheet node.  
  - Edge Cases: Incorrect URL field mapping; update failures.

- **Update row in sheet**  
  - Type: Google Sheets  
  - Role: Updates the corresponding row in the sheet with the "enviado_discord" flag and URL.  
  - Config: Matches rows on "url", updates "url" and "enviado_discord" columns.  
  - Inputs: Edit Fields1 output.  
  - Outputs: None  
  - Edge Cases: API errors; row not found if URL mismatch.

---

#### 2.5 AI Processing Block

**Overview:**  
Processes extracted PDF text using an AI language model to generate an executive summary.

**Nodes Involved:**  
- AI Agent  
- Simple Memory  
- OpenRouter Chat Model  
- Filtro caracteres max

**Node Details:**

- **AI Agent**  
  - Type: LangChain AI Agent  
  - Role: Analyzes the extracted text and generates an executive summary with key financial info, actions, board changes, and investor impact.  
  - Config: Prompt defines detailed instructions for summary; text input is `{{ $json.text }}` from PDF extraction.  
  - Inputs: Extract from File node output (text). Also connected to Simple Memory and OpenRouter Chat Model for memory and model input.  
  - Outputs: Filtro caracteres max node for message chunking.  
  - Edge Cases: Text input too long; prompt errors; model API issues.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context/session for the AI agent using a custom key "playway-agent".  
  - Config: Session key "playway-agent", session ID type customKey.  
  - Inputs: Connects to AI Agent’s memory input.  
  - Outputs: AI Agent node.  
  - Edge Cases: Memory overflow or reset issues.

- **OpenRouter Chat Model**  
  - Type: LangChain Chat Model  
  - Role: Provides the underlying AI language model API connection to OpenRouter using the "meta-llama/llama-4-maverick" free model.  
  - Config: Model selected from OpenRouter; API key required (not shown in workflow JSON).  
  - Inputs: AI Agent's languageModel input.  
  - Outputs: AI Agent node.  
  - Edge Cases: Authentication errors, rate limits, model availability.

- **Filtro caracteres max**  
  - Type: Code (JavaScript)  
  - Role: Splits AI-generated messages into chunks of max 1900 characters to comply with Discord message size limits.  
  - Config: Iterates over AI Agent output text, slices into 1900-character pieces, outputs array of message chunks.  
  - Inputs: AI Agent output.  
  - Outputs: Discord node.  
  - Edge Cases: Text smaller than chunk size; empty messages.

---

#### 2.6 Discord Notification

**Overview:**  
Sends the processed AI summaries or messages to a Discord channel via a webhook.

**Nodes Involved:**  
- Discord

**Node Details:**

- **Discord**  
  - Type: Discord node  
  - Role: Sends messages to a Discord channel using webhook authentication.  
  - Config: Content set to `{{ $json.message }}` from filtered chunks; webhook authentication used; webhook ID specified.  
  - Inputs: Filtro caracteres max output (message chunks).  
  - Outputs: None  
  - Edge Cases: Webhook invalid or deleted; message rate limiting; Discord API errors.

---

#### 2.7 Auxiliary Logic and Notes

**Overview:**  
Additional nodes for merging data, notes, and comments explaining configuration and usage.

**Nodes Involved:**  
- Sticky Note (multiple)  
- Other utility nodes integrated in previous blocks

**Node Details:**  
Sticky Notes provide guidance on configuration steps, such as setting the website URL, Google Cloud API credentials, OpenRouter API key, Discord webhook, and execution schedule. They do not directly process data but are critical for user understanding and maintenance.

---

### 3. Summary Table

| Node Name                    | Node Type                             | Functional Role                        | Input Node(s)                            | Output Node(s)                        | Sticky Note                                                                                   |
|------------------------------|-------------------------------------|-------------------------------------|-----------------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                      | Manual workflow start                | None                                    | HTTP Request                        | Start trigger block - manual start                                                           |
| Schedule Trigger             | Schedule Trigger                    | Weekly scheduled start               | None                                    | HTTP Request                        | Start trigger block - scheduled weekly execution                                             |
| HTTP Request                | HTTP Request                       | Fetch target website HTML            | When clicking ‘Execute workflow’, Schedule Trigger | HTML                              | Set the URL of the website to extract PDFs from                                             |
| HTML                       | HTML Extractor                     | Extract PDF links and titles         | HTTP Request                            | Code                               | Extract PDF links using CSS selector                                                        |
| Code                       | Code                              | Build full PDF URLs                  | HTML                                   | Get row(s) in sheet, Merge          | Prepends base URL to relative PDF hrefs                                                     |
| Get row(s) in sheet         | Google Sheets                     | Retrieve existing PDF URLs           | Code                                   | Merge                              | Connect to Google Sheets API; obtain existing data                                          |
| Merge                      | Merge                             | Combine new and existing URLs        | Get row(s) in sheet, Code               | Append or update row in sheet       | Merge based on URL to avoid duplicates                                                     |
| Append or update row in sheet | Google Sheets                     | Append or update PDF URL entries     | Merge                                  | HTTP Request1                      | Insert new URLs in sheet or update existing                                                |
| HTTP Request1               | HTTP Request                       | Download PDFs                       | Append or update row in sheet           | Extract from File                  | Download PDFs with retry and error continuation                                            |
| Extract from File           | Extract from File                  | Extract text from PDFs               | HTTP Request1                          | AI Agent, Edit Fields1              | Extract text content from PDFs                                                             |
| AI Agent                   | LangChain Agent                   | AI analysis and summarization        | Extract from File, Simple Memory, OpenRouter Chat Model | Filtro caracteres max             | Analyze report text and generate executive summary                                         |
| Simple Memory              | LangChain Memory                  | Maintain AI session memory           | AI Agent                              | AI Agent                          | Session management for AI agent                                                           |
| OpenRouter Chat Model      | LangChain Chat Model              | Connect to OpenRouter LLM            | AI Agent                              | AI Agent                          | Free model from OpenRouter; requires API key                                              |
| Filtro caracteres max       | Code                              | Chunk AI output messages             | AI Agent                              | Discord                          | Splits messages into 1900 char chunks for Discord limits                                  |
| Discord                    | Discord                           | Send messages to Discord channel     | Filtro caracteres max                  | None                             | Requires Discord webhook integration                                                     |
| Edit Fields1               | Set                              | Mark processed URLs with flag        | Extract from File                      | Update row in sheet               | Sets "enviado_discord" flag to "true"                                                    |
| Update row in sheet        | Google Sheets                     | Update sheet rows with processed flag | Edit Fields1                         | None                             | Updates rows matching on URL                                                             |
| Sticky Note (multiple)      | Sticky Note                      | Documentation and guidance           | None                                    | None                              | Various notes explaining setup and configuration (Google, Discord, OpenRouter, scheduling) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’" with default settings.  
   - Add a **Schedule Trigger** node named "Schedule Trigger" with interval set to 1 week.

2. **Set Up Website Access**  
   - Add an **HTTP Request** node named "HTTP Request".  
   - Configure the URL parameter with the target website URL containing PDFs.  
   - Connect both trigger nodes to this HTTP Request node.

3. **Extract PDF Links**  
   - Add an **HTML** node named "HTML".  
   - Set operation to "extractHtmlContent".  
   - Add extraction values:  
     - Key: "pdf", CSS selector: `a[href$=".pdf"]`, attribute: "href", returnArray: true.  
     - Key: "tittle", CSS selector: `a[href$=".pdf"]`, returnArray: true.  
   - Connect HTTP Request node to this HTML node.

4. **Build Complete PDF URLs**  
   - Add a **Code** node named "Code".  
   - Use the JavaScript code snippet to prepend the base URL (e.g., `https://www.playway.com`) to each relative PDF link.  
   - Connect HTML node output to this Code node.

5. **Configure Google Sheets - Get Existing Rows**  
   - Add a **Google Sheets** node named "Get row(s) in sheet".  
   - Set operation to "get rows" from the sheet "Sheet1" in the desired Google Sheets document (provide Document ID).  
   - Connect Code node output to this node (for execution order).

6. **Merge New and Existing URLs**  
   - Add a **Merge** node named "Merge".  
   - Set mode to "combine", join mode "keepNonMatches", matching on the "url" field.  
   - Connect "Get row(s) in sheet" (main output) and "Code" (main output) to the Merge node inputs.

7. **Append or Update URLs in Sheet**  
   - Add a **Google Sheets** node named "Append or update row in sheet".  
   - Configure to append or update rows in "PDFs" sheet, with columns "url" and "enviado_discord".  
   - Set matching column to "url".  
   - Connect Merge node output to this node.

8. **Download PDFs**  
   - Add an **HTTP Request** node named "HTTP Request1".  
   - Set URL to `={{ $json.url }}`.  
   - Configure response format as "file".  
   - Enable retry on failure and set "never error" to true for response.  
   - Connect "Append or update row in sheet" output to this node.

9. **Extract Text from PDFs**  
   - Add an **Extract from File** node named "Extract from File".  
   - Set operation to "pdf".  
   - Connect "HTTP Request1" output to this node.

10. **AI Processing Setup**  
    - Add an **OpenRouter Chat Model** node named "OpenRouter Chat Model".  
    - Select the "meta-llama/llama-4-maverick" model or another free model.  
    - Provide your OpenRouter API key in credentials.  

    - Add a **Simple Memory** node named "Simple Memory".  
    - Set sessionKey to "playway-agent" and sessionIdType to customKey.

    - Add an **AI Agent** node named "AI Agent".  
    - Set prompt text to analyze the company report with instructions to produce an executive summary (use the provided prompt in the workflow).  
    - Connect "Extract from File" output (text) to AI Agent’s text input.  
    - Connect OpenRouter Chat Model to AI Agent’s languageModel input.  
    - Connect Simple Memory to AI Agent’s ai_memory input.

11. **Chunk AI Output Messages**  
    - Add a **Code** node named "Filtro caracteres max".  
    - Use the JavaScript code to split messages into 1900-character chunks for Discord.  
    - Connect AI Agent output to this node.

12. **Send to Discord**  
    - Add a **Discord** node named "Discord".  
    - Configure to send messages via webhook authentication.  
    - Paste your Discord webhook URL or select credentials.  
    - Set content to `={{ $json.message }}`.  
    - Connect "Filtro caracteres max" output to this node.

13. **Mark URLs as Processed**  
    - Add a **Set** node named "Edit Fields1".  
    - Set "url" to `={{ $('HTTP Request1').item.json.url }}` and "enviado_discord" to "true".  
    - Connect "Extract from File" output to this node.

    - Add a **Google Sheets** node named "Update row in sheet".  
    - Configure to update rows in the "PDFs" sheet, matching by "url", updating "url" and "enviado_discord" columns.  
    - Connect "Edit Fields1" output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                   |
|------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| To use Google Sheets integration, enable Google Drive and Sheets APIs in Google Cloud Console.       | https://cloud.google.com/                          |
| For AI processing, register at OpenRouter and obtain a free API key; select any free LLM model.     | https://openrouter.ai/                             |
| Create a private Discord server and channel; set up webhook integration to receive messages.         | Discord webhook setup in channel settings         |
| The website URL in the initial HTTP Request node must be set to the site where PDFs are hosted.      | Replace placeholder URL with target website URL  |
| Schedule Trigger node runs the workflow weekly, but can be adjusted to other intervals as needed.    | n8n Schedule Trigger documentation                 |
| The workflow uses a base URL hardcoded in the Code node; adjust if the website structure changes.     | Modify in Code node JavaScript code                |
| Slack or email notifications can be added by extending the workflow after the AI Agent node.         | Custom extension possibility                       |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.