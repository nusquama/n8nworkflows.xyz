Newsletter Summarization & Briefing with Gmail, AI, Google Sheets & Email

https://n8nworkflows.xyz/workflows/newsletter-summarization---briefing-with-gmail--ai--google-sheets---email-9633


# Newsletter Summarization & Briefing with Gmail, AI, Google Sheets & Email

---
### 1. Workflow Overview

This workflow, titled **Newsletter Summarization & Briefing with Gmail, AI, Google Sheets & Email**, automates the process of collecting daily newsletters from Gmail, cleaning and organizing their content, summarizing them using a large language model (LLM), and delivering a structured briefing email. It is designed primarily for Public Affairs professionals but is adaptable to other domains by modifying prompts and filters.

The workflow is logically divided into the following blocks:

**1.1 Scheduling and Data Collection**  
- Triggers daily at a specified hour.  
- Creates or updates a Google Spreadsheet to store raw newsletter text and metadata.  
- Fetches newsletters from Gmail inbox.

**1.2 Content Cleaning and Chunking**  
- Parses and sanitizes HTML content from raw emails to clean plain text.  
- Splits long newsletter texts into manageable chunks to ensure token-safe processing.

**1.3 Data Logging**  
- Maps cleaned and chunked data into structured fields.  
- Appends rows of cleaned data to the Google Spreadsheet for record-keeping.

**1.4 AI Processing and Analysis**  
- Uses Google Gemini Chat LLM with memory buffer to analyze collected newsletters.  
- Generates a comprehensive briefing report in German, including executive summary, major themes, stakeholder analysis, strategic implications, recommended actions, and open questions.

**1.5 Email Generation and Delivery**  
- Converts the AI-generated briefing from Markdown-like format into styled HTML email.  
- Sends the formatted briefing email to defined recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Data Collection

**Overview:**  
This block schedules the workflow daily and initializes data storage by creating or updating a Google Spreadsheet. It then fetches all newsletters from Gmail.

**Nodes Involved:**  
- Schedule Trigger  
- Google Spreadsheet  
- Gmail

**Node Details:**

- **Schedule Trigger**  
  - Type: `n8n-nodes-base.scheduleTrigger`  
  - Role: Triggers the workflow daily at 9:00 AM.  
  - Configuration: Set to run once daily at hour 9.  
  - Inputs: None  
  - Outputs: Google Spreadsheet node  
  - Edge Cases: Workflow will not run outside scheduled time; ensure timezone correctness.

- **Google Spreadsheet**  
  - Type: `n8n-nodes-base.googleSheets`  
  - Role: Creates or updates a spreadsheet named with the current date (yy/mm/dd format) and sheet named "Inbox".  
  - Configuration: Uses dynamic title based on current date.  
  - Inputs: Schedule Trigger  
  - Outputs: Gmail  
  - Edge Cases: Requires valid Google Sheets credentials and API access; handle rate limits and quota.

- **Gmail**  
  - Type: `n8n-nodes-base.gmail`  
  - Role: Retrieves all emails (default filter, no label specified) from Gmail inbox.  
  - Configuration: Operation "getAll", no label filter set (can be customized).  
  - Inputs: Google Spreadsheet  
  - Outputs: Parse HTML-Code  
  - Edge Cases: Requires OAuth2 credentials for Gmail; potential API quota limits; may retrieve unwanted emails if filters are not set.

---

#### 2.2 Content Cleaning and Chunking

**Overview:**  
This block cleans HTML content from emails by removing clutter and boilerplate, then splits long texts into token-safe chunks for processing.

**Nodes Involved:**  
- Parse HTML-Code  
- Split in Chunks

**Node Details:**

- **Parse HTML-Code**  
  - Type: `n8n-nodes-base.code` (JavaScript)  
  - Role: Cleans HTML email bodies to plain text by removing HTML tags, scripts, styles, email addresses, tracking URLs, boilerplate footers, and other noise.  
  - Key Code Elements:  
    - Removes RTF artifacts, HTML tags, comments, scripts, styles.  
    - Normalizes whitespace and special characters.  
    - Filters out common boilerplate phrases and footer markers aggressively.  
    - Removes tracking/unsubscribe links and email addresses.  
    - Compresses multiple blank lines and protects Excel formula injection.  
  - Inputs: Gmail node (raw emails)  
  - Outputs: Split in Chunks node  
  - Edge Cases: Complex HTML may cause incomplete cleaning; aggressive boilerplate removal might discard relevant text; malformed HTML may cause parsing errors.

- **Split in Chunks**  
  - Type: `n8n-nodes-base.code` (JavaScript)  
  - Role: Splits cleaned newsletter text into chunks of maximum 50,000 characters, attempting to cut at paragraph or sentence boundaries to avoid mid-word splits.  
  - Configuration: Max length 50,000 characters; splits on double newlines or period+space near cut points.  
  - Inputs: Parse HTML-Code  
  - Outputs: Set Table Fields  
  - Edge Cases: Very long newsletters may produce multiple chunks; chunk boundaries may sometimes split content awkwardly if no clear break found.

---

#### 2.3 Data Logging

**Overview:**  
This block maps the chunked data to structured fields and appends each as a row in the Google Spreadsheet.

**Nodes Involved:**  
- Set Table Fields  
- Append Row

**Node Details:**

- **Set Table Fields**  
  - Type: `n8n-nodes-base.set`  
  - Role: Maps newsletter metadata and text chunk into explicit spreadsheet columns: sender address, sender name, subject, text.  
  - Configuration: Fields assigned from input JSON keys.  
  - Inputs: Split in Chunks  
  - Outputs: Append Row  
  - Edge Cases: Missing metadata fields could cause empty columns.

- **Append Row**  
  - Type: `n8n-nodes-base.googleSheets`  
  - Role: Appends the structured data rows into the spreadsheet's "Inbox" sheet.  
  - Configuration: Auto-maps input data to sheet columns; uses spreadsheet ID and sheet name dynamically from the Google Spreadsheet node.  
  - Inputs: Set Table Fields  
  - Outputs: Public Affairs Consultant  
  - Edge Cases: Requires valid Google Sheets credentials; append failure if sheet or document inaccessible.

---

#### 2.4 AI Processing and Analysis

**Overview:**  
This block aggregates all newsletter chunks and uses an LLM (Google Gemini Chat Model) with memory buffer to generate a detailed Public Affairs briefing in German.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Memory3  
- Public Affairs Consultant

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  - Role: Provides the AI language model interface for chat-based interaction.  
  - Configuration: Default options, integrated with LangChain.  
  - Inputs: None (AI input via Public Affairs Consultant)  
  - Outputs: Public Affairs Consultant (AI response)  
  - Edge Cases: Requires valid Google Gemini API credentials; potential API limits or latency.

- **Memory3**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Maintains a sliding window memory buffer of last 10 interactions with key "newsletter_briefing_2025".  
  - Configuration: Custom session key, window length 10.  
  - Inputs: None (connected as AI memory to Public Affairs Consultant)  
  - Outputs: Public Affairs Consultant (memory context)  
  - Edge Cases: Memory overflow or inconsistency if session key collides; memory resets if workflow restarts.

- **Public Affairs Consultant**  
  - Type: `@n8n/n8n-nodes-langchain.code`  
  - Role: Core logic node invoking the LLM chain to produce a deep-dive briefing report in German.  
  - Configuration:  
    - Aggregates all newsletter chunks into a single formatted Markdown text.  
    - Uses a crafted prompt template to instruct the LLM to analyze, synthesize themes, stakeholder positions, strategic implications, and recommended actions.  
    - Enforces German language output and JSON format for structured data extraction.  
    - Returns a JSON object with fields like executiveSummary, majorThemes, keyDevelopments, stakeholderAnalysis, crossCuttingInsights, strategicImplications, recommendedActions, gapsAndQuestions, and metadata.  
  - Inputs: Memory3 (memory), Google Gemini Chat Model (language model), Append Row (newsletter data)  
  - Outputs: Structure HTML-Mail  
  - Edge Cases: LLM failure or timeout; JSON parse errors; incomplete or malformed AI output; language compliance issues.

---

#### 2.5 Email Generation and Delivery

**Overview:**  
This block formats the AI briefing report into a styled HTML email and sends it to specified recipients.

**Nodes Involved:**  
- Structure HTML-Mail  
- Send email

**Node Details:**

- **Structure HTML-Mail**  
  - Type: `n8n-nodes-base.code` (JavaScript)  
  - Role: Converts JSON briefing data with markdown-like fields to a visually styled HTML email template.  
  - Key Features:  
    - Markdown to HTML conversion for bold text, lists, paragraphs, and line breaks.  
    - Styled sections for executive summary, major themes, stakeholder analysis, strategic insights, recommended actions, and open questions.  
    - Responsive design with consistent fonts, colors, and spacing.  
    - Metadata footer noting source count and automated generation.  
  - Inputs: Public Affairs Consultant  
  - Outputs: Send email  
  - Edge Cases: Malformed input JSON or missing fields could break HTML formatting.

- **Send email**  
  - Type: `n8n-nodes-base.emailSend`  
  - Role: Sends the generated HTML email to designated recipients.  
  - Configuration:  
    - Subject dynamically generated with German date and "Newsletter Zusammenfassung von <date>".  
    - Recipient email: "recipient@example.com" (placeholder).  
    - From email: "sender@example.com" (placeholder).  
    - Email body set to generated HTML content.  
  - Inputs: Structure HTML-Mail  
  - Outputs: None (end node)  
  - Edge Cases: SMTP credential issues; network errors; invalid email addresses.

---

### 3. Summary Table

| Node Name             | Node Type                                | Functional Role                            | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                      |
|-----------------------|----------------------------------------|--------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | n8n-nodes-base.scheduleTrigger         | Triggers workflow daily at 9 AM            | None                       | Google Spreadsheet             | Group A – Timing & Destination: Runs daily; creates/updates Google Sheet. Customize interval/time & sheet name. |
| Google Spreadsheet     | n8n-nodes-base.googleSheets             | Creates/updates spreadsheet to store data | Schedule Trigger           | Gmail                         | Group A – Timing & Destination                                                                                   |
| Gmail                 | n8n-nodes-base.gmail                    | Fetches newsletters from Gmail inbox       | Google Spreadsheet         | Parse HTML-Code                | Inbox, Pre-Clean & Chunking: Pulls newsletters via Gmail query; removes HTML clutter. Customize query/filters.   |
| Parse HTML-Code        | n8n-nodes-base.code                     | Cleans HTML content to plain text           | Gmail                      | Split in Chunks               | Inbox, Pre-Clean & Chunking                                                                                      |
| Split in Chunks        | n8n-nodes-base.code                     | Splits long texts into token-safe chunks   | Parse HTML-Code             | Set Table Fields              | Inbox, Pre-Clean & Chunking                                                                                      |
| Set Table Fields       | n8n-nodes-base.set                      | Maps chunked data to spreadsheet fields    | Split in Chunks             | Append Row                   | Inbox, Pre-Clean & Chunking                                                                                      |
| Append Row             | n8n-nodes-base.googleSheets             | Appends data rows to Google Sheet           | Set Table Fields            | Public Affairs Consultant      | Inbox, Pre-Clean & Chunking                                                                                      |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides AI language model                   | None                       | Public Affairs Consultant      | LLM Policy Digest: AI summarization and briefing. Customize prompt, memory scope.                                |
| Memory3                | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains AI memory buffer                    | None                       | Public Affairs Consultant      | LLM Policy Digest                                                                                                |
| Public Affairs Consultant | @n8n/n8n-nodes-langchain.code          | Aggregates newsletters & generates briefing | Append Row, Memory3, Gemini | Structure HTML-Mail            | LLM Policy Digest                                                                                                |
| Structure HTML-Mail    | n8n-nodes-base.code                     | Converts briefing JSON to styled HTML email | Public Affairs Consultant   | Send email                   | Build & Send Email: Generates email with briefing; customizable recipients & branding.                           |
| Send email             | n8n-nodes-base.emailSend                | Sends briefing email                         | Structure HTML-Mail         | None                         | Build & Send Email                                                                                                |
| Sticky Note            | n8n-nodes-base.stickyNote               | Overview comments                            | None                       | None                         | Overview: Workflow steps summarized; adaptable beyond Public Affairs.                                           |
| Sticky Note1           | n8n-nodes-base.stickyNote               | Scheduling and destination comments          | None                       | None                         | Group A – Timing & Destination                                                                                    |
| Sticky Note2           | n8n-nodes-base.stickyNote               | Inbox and cleaning comments                   | None                       | None                         | Inbox, Pre-Clean & Chunking                                                                                      |
| Sticky Note3           | n8n-nodes-base.stickyNote               | LLM summarization comments                    | None                       | None                         | LLM Policy Digest                                                                                                |
| Sticky Note4           | n8n-nodes-base.stickyNote               | Email build and send comments                 | None                       | None                         | Build & Send Email                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to run daily at 9:00 AM (hour=9)  
   - No inputs, connects to Google Spreadsheet node

2. **Create a Google Spreadsheet node**  
   - Type: Google Sheets  
   - Operation: Create or update spreadsheet  
   - Title: Set dynamically as current date in `yy/mm/dd` format using expression:  
     `={{ new Date().toISOString().slice(2,10).replace(/-/g, '/') }}`  
   - Sheet name tab: "Inbox" (create if missing)  
   - Connect input from Schedule Trigger  
   - Connect output to Gmail node

3. **Create a Gmail node**  
   - Operation: Get all emails (`getAll`)  
   - Filters: None by default; customize with query or label to target newsletters (e.g., label:PA-News newer_than:1d)  
   - OAuth2 credentials required  
   - Connect input from Google Spreadsheet  
   - Connect output to Parse HTML-Code

4. **Create a Code node named "Parse HTML-Code"**  
   - Type: Code (JavaScript)  
   - Function: Clean HTML content of emails into plain text, removing scripts, styles, email addresses, tracking links, boilerplate, and compressing whitespace.  
   - Use the provided cleaning script logic (see Node 2.2)  
   - Run once per email item  
   - Input from Gmail  
   - Output to Split in Chunks

5. **Create a Code node named "Split in Chunks"**  
   - Type: Code (JavaScript)  
   - Function: Split cleaned text into maximum 50,000 character chunks, cutting preferably at paragraph or sentence boundaries.  
   - Input from Parse HTML-Code  
   - Output to Set Table Fields

6. **Create a Set node named "Set Table Fields"**  
   - Type: Set  
   - Map fields:  
     - `Absenderadresse` = `{{$json.Absenderadresse}}`  
     - `Absendername` = `{{$json.Absendername}}`  
     - `Betreff` = `{{$json.Betreff}}`  
     - `Text` = `{{$json.Text}}`  
   - Input from Split in Chunks  
   - Output to Append Row

7. **Create a Google Sheets node named "Append Row"**  
   - Operation: Append data to sheet  
   - Sheet: Use spreadsheet ID and sheet name dynamically from the previous Google Spreadsheet node outputs  
   - Columns mapped automatically from Set Table Fields  
   - Input from Set Table Fields  
   - Output to Public Affairs Consultant

8. **Create a Google Gemini Chat Model node**  
   - Type: LangChain Google Gemini Chat Model  
   - Provide credentials for Google Gemini API  
   - No inputs; will connect to Public Affairs Consultant as AI language model

9. **Create a Memory Buffer Window node named "Memory3"**  
   - Type: LangChain memory buffer window  
   - Session key: "newsletter_briefing_2025"  
   - Context window length: 10  
   - No direct input; connected as AI memory input for Public Affairs Consultant

10. **Create a LangChain Code node named "Public Affairs Consultant"**  
    - Type: LangChain code  
    - Inputs:  
      - Main input from Append Row (newsletter data)  
      - AI language model input from Google Gemini Chat Model  
      - AI memory input from Memory3  
    - Function: Aggregate all newsletter chunks into a single Markdown-formatted text; run a prompt template instructing the LLM to create a detailed German briefing report with structured JSON output (see Node 2.4 for prompt details).  
    - Output to Structure HTML-Mail

11. **Create a Code node named "Structure HTML-Mail"**  
    - Type: Code (JavaScript)  
    - Function: Convert the briefing JSON fields (which include markdown) into a styled HTML email template with CSS formatting for headings, paragraphs, lists, boxes, and sections.  
    - Input from Public Affairs Consultant  
    - Output to Send email

12. **Create an Email Send node named "Send email"**  
    - Type: Email Send  
    - From Email: Set your sender email (e.g., "sender@example.com")  
    - To Email: Set recipient email (e.g., "recipient@example.com")  
    - Subject: Dynamic expression generating German date string, e.g.:  
      `={{ "Newsletter Zusammenfassung von " + $now.toLocaleString("de-DE", { weekday: "long", day: "2-digit", month: "long", year: "numeric" }).replace(/^\w/, c => c.toUpperCase()) }}`  
    - Body: Use HTML from Structure HTML-Mail node output  
    - SMTP or other email credentials configured  
    - Input from Structure HTML-Mail

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This template can be adapted beyond Public Affairs by modifying the AI prompt and Gmail filters.    | Sticky Note (Overview)                                                                                |
| Customize Gmail query to fetch relevant newsletters only, e.g., label filters or date restrictions. | Sticky Note2 (Inbox, Pre-Clean & Chunking)                                                          |
| The AI prompt enforces German language output and structured JSON for downstream processing.       | Sticky Note3 (LLM Policy Digest)                                                                     |
| The HTML email template is styled with CSS for readability and branding; you can customize styles.  | Sticky Note4 (Build & Send Email)                                                                    |
| Requires Google API credentials (Sheets and Gemini) and Gmail OAuth2 credentials for full operation.| Setup notes for credentials                                                                            |
| Potential failure points include API rate limits, malformed emails, LLM timeouts, and email send errors. | General workflow considerations                                                                      |

---

**Disclaimer:** The provided content exclusively derives from an automated n8n workflow. It strictly complies with content policies and contains no illegal or offensive material. All processed data is legal and public.