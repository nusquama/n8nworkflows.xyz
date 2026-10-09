Create daily construction site reports from WhatsApp with OpenAI and Google

https://n8nworkflows.xyz/workflows/create-daily-construction-site-reports-from-whatsapp-with-openai-and-google-20556


# Create daily construction site reports from WhatsApp with OpenAI and Google

### 1. Workflow Overview

This workflow automates the collection, processing, and reporting of construction site updates received via WhatsApp. It bridges field communications (text, voice notes, and photos) with backend project management systems (Google Sheets, Google Drive, and Gmail) using OpenAI for transcription, image analysis, and data structuring. 

The workflow operates via two distinct triggers:
1. **Real-Time WhatsApp Intake Branch:** Captures inbound messages from construction crews, verifies sender registration, processes media attachments (audio transcription, image analysis, and Google Drive storage), extracts structured site-log metrics using an LLM, logs them to Google Sheets, replies to the sender, and alerts project managers if critical issues or safety concerns arise.
2. **Scheduled Evening Reporting Branch:** Executes daily at 6:00 PM, aggregates the day's site logs by project, generates comprehensive AI-driven daily status reports (or fallback "no-update" notices), emails them to designated project stakeholders, and archives summary records for tracking history.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
- **Overview:** Sets up global configuration parameters and routes execution depending on whether a WhatsApp message arrived or the 6:00 PM schedule was triggered.
- **Nodes Involved:** `When WhatsApp Message Received`, `Every Evening at 6pm`, `Set Configuration Parameters`, `Route by Trigger Event`.
- **Node Details:**
  - `When WhatsApp Message Received` (n8n-nodes-base.whatsAppTrigger): 
    - *Role:* Entry point for inbound WhatsApp webhooks (`messages`).
    - *Config:* Uses WhatsApp Trigger credentials and listens for incoming message payloads.
    - *Input/Output:* Inputs none; outputs raw webhook payload to `Set Configuration Parameters`.
    - *Errors:* Webhook verification failures or missing Meta API tokens.
  - `Every Evening at 6pm` (n8n-nodes-base.scheduleTrigger):
    - *Role:* Scheduled entry point for end-of-day reports.
    - *Config:* Rule set to trigger daily at hour 18:00.
    - *Input/Output:* Inputs none; outputs schedule execution event to `Set Configuration Parameters`.
  - `Set Configuration Parameters` (n8n-nodes-base.set):
    - *Role:* Establishes centralized environment variables (company name, Google Sheet URL, Google Drive folder ID, office fallback email, and report language).
    - *Config:* Assigns string variables to the execution context.
    - *Input/Output:* Receives input from either trigger; outputs configuration data.
  - `Route by Trigger Event` (n8n-nodes-base.switch):
    - *Role:* Determines whether the workflow runs the WhatsApp intake path or the evening reporting path.
    - *Config:* Switch node evaluating `messages` array presence vs. the schedule trigger execution state.
    - *Input/Output:* Input from `Set Configuration Parameters`; outputs to `Read Crew List from Sheets` (WhatsApp path) or `Read Site Log for Today` (Daily report path).

#### 1.2 Sender & Context Resolution
- **Overview:** Validates inbound WhatsApp messages against active crew rosters and project databases, resolving sender identity, assigned projects, and custom `#ProjectName` tags.
- **Nodes Involved:** `Read Crew List from Sheets`, `Read Project List from Sheets`, `Identify Sender and Project`, `Check if Registered Crew Member`, `Reply: Not a Registered Number`.
- **Node Details:**
  - `Read Crew List from Sheets` (n8n-nodes-base.googleSheets):
    - *Role:* Loads the authorized crew member directory from Google Sheets.
    - *Config:* Operation: `read`, Sheet Name: `Crew`, Document ID dynamic expression pointing to `sheetUrl`.
    - *Input/Output:* Input from `Route by Trigger Event`; outputs crew row items.
    - *Errors:* Google Sheets API quota limits, missing sheet tabs, or invalid credentials.
  - `Read Project List from Sheets` (n8n-nodes-base.googleSheets):
    - *Role:* Loads active projects and report recipient lists.
    - *Config:* Operation: `read`, Sheet Name: `Projects`, Document ID dynamic expression. `executeOnce` is enabled.
    - *Input/Output:* Input from `Read Crew List from Sheets`; outputs project row items.
  - `Identify Sender and Project` (n8n-nodes-base.code):
    - *Role:* Normalizes phone numbers, matches senders against the crew list, parses text/captions for `#ProjectName` tags, and constructs a unified context object.
    - *Config:* Custom JavaScript processing incoming payloads, phone normalization, and regex hashtag matching.
    - *Input/Output:* Inputs from project sheets; outputs a consolidated message context JSON.
  - `Check if Registered Crew Member` (n8n-nodes-base.if):
    - *Role:* Branches execution based on whether the sender's phone number exists in the crew list.
    - *Config:* Evaluates boolean condition `{{ $json.registered }}`.
    - *Input/Output:* Input from code node; true branch goes to `Route by Message Type`, false branch goes to `Reply: Not a Registered Number`.
  - `Reply: Not a Registered Number` (n8n-nodes-base.whatsApp):
    - *Role:* Sends an automated rejection message via WhatsApp if an unregistered number attempts to log an update.
    - *Config:* Operation: `send`, uses WhatsApp credentials and dynamic recipient phone numbers.
    - *Input/Output:* Input from `Check if Registered Crew Member` (false); outputs message delivery receipt.
    - *Errors:* WhatsApp API messaging restrictions outside the 24-hour customer service window.

#### 1.3 Media & Message Handling
- **Overview:** Routes supported message formats (text, voice, image), downloads binary media assets, transcribes audio, describes images via computer vision, and saves photos to Google Drive.
- **Nodes Involved:** `Route by Message Type`, `Fetch Voice Note URL`, `Download Voice Note`, `OpenAI Transcription`, `Fetch Photo URL`, `Download Photo`, `Describe Photo Using AI`, `Save Photo to Google Drive`, `Merge Photo Descriptions`, `Reply: Unsupported Message Format`, `Normalize Update Into Text`.
- **Node Details:**
  - `Route by Message Type` (n8n-nodes-base.switch):
    - *Role:* Directs messages to audio, image, text, or fallback handling paths based on `type`.
    - *Config:* Switch rules matching `audio`, `image`, `text`, with fallback output set to `Other`.
    - *Input/Output:* Input from `Check if Registered Crew Member`; outputs to respective media download or reply nodes.
  - `Fetch Voice NoteURL` / `Download Voice Note` / `OpenAI Transcription`:
    - *Role:* Retrieves WhatsApp voice media metadata, downloads the binary audio file via HTTP request, and converts speech to text using OpenAI Audio Transcription.
    - *Config:* Uses WhatsApp API credentials for URL fetch/download, and OpenAI credentials for transcription.
    - *Errors:* Expired media URLs or unsupported audio formats.
  - `Fetch PhotoURL` / `Download Photo` / `Describe Photo Using AI` / `Save Photo to Google Drive` / `Merge Photo Descriptions`:
    - *Role:* Downloads site photos, generates a factual 2-to-4 sentence AI description using GPT-4o-mini, archives the original file to a designated Google Drive folder, and merges textual descriptions with drive file references.
    - *Config:* Utilizes OpenAI (`analyze` operation, base64 input) and Google Drive (`upload` operation with `continueRegularOutput` error handling).
    - *Errors:* Google Drive storage limits, folder permission issues, or rate limits.
  - `Reply: Unsupported Message Format` (n8n-nodes-base.whatsApp):
    - *Role:* Informs crew members via WhatsApp if they send unsupported message types (e.g., documents, stickers).
  - `Normalize Update Into Text` (n8n-nodes-base.code):
    - *Role:* Consolidates text updates, transcriptions, photo descriptions, and drive links into a single standardized text payload.

#### 1.4 LLM Extraction & Logging
- **Overview:** Processes normalized text updates through an LLM chain with structured output parsing to extract construction log metrics, builds a Google Sheets row, appends it, confirms receipt to the crew, and triggers PM alerts if necessary.
- **Nodes Involved:** `Extract Site Log Fields`, `OpenAI Model for Extraction`, `Site Log Output Format Parser`, `Build Log Row Data`, `Append to Site Log Sheet`, `Confirm Receipt to Crew`, `Check PM Attention Needed`, `Send PM Alert Email`.
- **Node Details:**
  - `Extract Site Log Fields` & supporting AI nodes (`OpenAI Model for Extraction`, `Site Log Output Format Parser`):
    - *Role:* Extracts structured fields (summary, category, work completed, crew count, hours, materials, issues, safety concerns, attention flags, weather) using GPT-4o-mini with a strict JSON schema and translation rules matching `reportLanguage`.
    - *Config:* Model temperature set to `0.1` for deterministic extraction; strict output schema enforcement.
    - *Input/Output:* Input from `Normalize Update Into Text`; outputs structured JSON to `Build Log Row Data`.
    - *Errors:* LLM JSON parsing errors or timeout issues during high API loads.
  - `Build Log Row Data` (n8n-nodes-base.code):
    - *Role:* Flattens the extracted AI schema and metadata into a clean, flat object mapped directly to Google Sheets column headers.
  - `Append to Site Log Sheet` (n8n-nodes-base.googleSheets):
    - *Role:* Appends the parsed log record to the `Site log` tab.
    - *Config:* Operation: `append`, Sheet Name: `Site log`, Cell Format: `RAW`.
  - `Confirm Receipt to Crew` (n8n-nodes-base.whatsApp):
    - *Role:* Sends a confirmation WhatsApp message to the reporter summarizing what was logged and noting any attention flags or unassigned project tips.
  - `Check PM Attention Needed` & `Send PM Alert Email` (n8n-nodes-base.if / n8n-nodes-base.gmail):
    - *Role:* Evaluates whether `Needs attention` equals `Yes`, and if so, dispatches a formatted HTML alert email via Gmail to the project's report email list.

#### 1.5 Scheduled Evening Reporting
- **Overview:** Aggregates daily site logs at 6:00 PM, groups updates by project, compiles end-of-day reports using OpenAI, emails them to stakeholders, and archives report metadata.
- **Nodes Involved:** `Read Site Log for Today`, `Read Projects for Reports`, `Group Updates by Project`, `Check for Todays Updates`, `Compile Daily Report`, `OpenAI Model for Reports`, `Daily Report Format Parser`, `Construct Report Email`, `Create No-Update Notice`, `Send Daily Report Email`, `Format Report Row`, `Append to Daily Reports Sheet`.
- **Node Details:**
  - `Read Site Log for Today` & `Read Projects for Reports` (n8n-nodes-base.googleSheets):
    - *Role:* Queries today's rows from the `Site log` sheet (filtered by current date) and reads the master `Projects` sheet to check active project states and recipient lists.
  - `Group Updates by Project` (n8n-nodes-base.code):
    - *Role:* Groups site logs by project name, calculates cumulative hours and crew counts, collects safety flags, and prepares aggregate datasets.
  - `Check for Todays Updates` (n8n-nodes-base.if):
    - *Role:* Splits execution paths between active projects that received updates (routing to AI report compilation) and active projects with zero updates (routing to no-update notices).
  - `Compile Daily Report` & AI nodes (`OpenAI Model for Reports`, `Daily Report Format Parser`):
    - *Role:* Uses GPT-4o-mini to write professional end-of-day project reports, determining overall status (`on_track`, `at_risk`, `blocked`), progress summaries, de-duplicated issues, and next steps.
  - `Construct Report Email` / `Create No-Update Notice` / `Send Daily Report Email`:
    - *Role:* Builds responsive HTML emails containing KPIs, timelines, photo links, and status badges, then sends them via Gmail to project stakeholders.
  - `Format Report Row` & `Append to Daily Reports Sheet` (n8n-nodes-base.set / n8n-nodes-base.googleSheets):
    - *Role:* Formats execution summaries into row objects and appends them to the `Daily reports` Google Sheet tab for historical tracking.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow overview & setup instructions | None | None | ## Turn WhatsApp voice notes and photos from job sites into daily construction reports... |
| Sheet setup | n8n-nodes-base.stickyNote | Google Sheet schema reference | None | None | ## Sheet setup Create one Google Sheet with these tabs and header rows... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Intake group label | None | None | ## Receive WhatsApp updates Entry point for job-site messages sent by crew members through WhatsApp. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Reporting group label | None | None | ## Start evening reports Scheduled trigger that starts the daily report generation flow at 6pm. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Configuration group label | None | None | ## Load shared settings Central configuration and trigger router... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Context resolution group label | None | None | ## Resolve sender context Looks up crew and project records... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Validation group label | None | None | ## Validate and route message Checks whether the sender is registered... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Transcription group label | None | None | ## Transcribe voice notes Fetches the WhatsApp voice note media URL... |
| Sticky Note7 | n8n-nodes-base.stickyNote | Photo processing group label | None | None | ## Process site photos Downloads WhatsApp photo media, generates an AI description... |
| Sticky Note8 | n8n-nodes-base.stickyNote | LLM extraction group label | None | None | ## Extract log fields Normalizes text, voice, or photo inputs into a plain-text update... |
| Sticky Note9 | n8n-nodes-base.stickyNote | Logging & alerting group label | None | None | ## Save log and alert Appends the structured update to the site log... |
| Sticky Note10 | n8n-nodes-base.stickyNote | Evening collection group label | None | None | ## Collect report inputs For the scheduled branch, retrieves today’s site-log rows... |
| Sticky Note11 | n8n-nodes-base.stickyNote | Evening composition group label | None | None | ## Compose report content Uses an LLM and structured output format to write daily reports... |
| Sticky Note12 | n8n-nodes-base.stickyNote | Evening archive group label | None | None | ## Send and archive reports Emails the daily project report or no-update notice... |
| When WhatsApp Message Received | n8n-nodes-base.whatsAppTrigger | Inbound WhatsApp webhook trigger | None | Set Configuration Parameters | ## Receive WhatsApp updates... |
| Every Evening at 6pm | n8n-nodes-base.scheduleTrigger | Daily 6:00 PM schedule trigger | None | Set Configuration Parameters | ## Start evening reports... |
| Set Configuration Parameters | n8n-nodes-base.set | Assigns global environment variables | When WhatsApp Message Received, Every Evening at 6pm | Route by Trigger Event | ## Load shared settings... |
| Route by Trigger Event | n8n-nodes-base.switch | Routes execution by trigger source | Set Configuration Parameters | Read Crew List from Sheets, Read Site Log for Today | ## Load shared settings... |
| Read Crew List from Sheets | n8n-nodes-base.googleSheets | Reads authorized crew directory | Route by Trigger Event | Read Project List from Sheets | ## Resolve sender context... |
| Read Project List from Sheets | n8n-nodes-base.googleSheets | Reads project list and email lists | Read Crew List from Sheets | Identify Sender and Project | ## Resolve sender context... |
| Identify Sender and Project | n8n-nodes-base.code | Matches sender, project, and tags | Read Project List from Sheets | Check if Registered Crew Member | ## Resolve sender context... |
| Check if Registered Crew Member | n8n-nodes-base.if | Validates crew registration status | Identify Sender and Project | Route by Message Type, Reply: Not a Registered Number | ## Validate and route message... |
| Reply: Not a Registered Number | n8n-nodes-base.whatsApp | Sends unregistered sender warning | Check if Registered Crew Member | None | ## Validate and route message... |
| Route by Message Type | n8n-nodes-base.switch | Routes by message media type | Check if Registered Crew Member | Fetch Voice Note URL, Fetch Photo URL, Normalize Update Into Text, Reply: Unsupported Message Format | ## Validate and route message... |
| Fetch Voice Note URL | n8n-nodes-base.whatsApp | Gets audio media download URL | Route by Message Type | Download Voice Note | ## Transcribe voice notes... |
| Download Voice Note | n8n-nodes-base.httpRequest | Downloads audio binary file | Fetch Voice Note URL | OpenAI Transcription | ## Transcribe voice notes... |
| OpenAI Transcription | @n8n/n8n-nodes-langchain.openAi | Transcribes audio to text | Download Voice Note | Normalize Update Into Text | ## Transcribe voice notes... |
| Fetch Photo URL | n8n-nodes-base.whatsApp | Gets image media download URL | Route by Message Type | Download Photo | ## Process site photos... |
| Download Photo | n8n-nodes-base.httpRequest | Downloads image binary file | Fetch Photo URL | Describe Photo Using AI, Save Photo to Google Drive | ## Process site photos... |
| Describe Photo Using AI | @n8n/n8n-nodes-langchain.openAi | Generates image description | Download Photo | Merge Photo Descriptions | ## Process site photos... |
| Save Photo to Google Drive | n8n-nodes-base.googleDrive | Uploads photo to Google Drive | Download Photo | Merge Photo Descriptions | ## Process site photos... |
| Merge Photo Descriptions | n8n-nodes-base.merge | Combines photo analysis and drive link | Describe Photo Using AI, Save Photo to Google Drive | Normalize Update Into Text | ## Process site photos... |
| Reply: Unsupported Message Format | n8n-nodes-base.whatsApp | Replies to unsupported message types | Route by Message Type | None | ## Validate and route message... |
| Normalize Update Into Text | n8n-nodes-base.code | Normalizes inputs into plain text | OpenAI Transcription, Merge Photo Descriptions, Route by Message Type | Extract Site Log Fields | ## Extract log fields... |
| Extract Site Log Fields | @n8n/n8n-nodes-langchain.chainLlm | Extracts site log attributes via LLM | Normalize Update Into Text | Build Log Row Data | ## Extract log fields... |
| OpenAI Model for Extraction | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for extraction (GPT-4o-mini) | None (AI sub-node) | Extract Site Log Fields | ## Extract log fields... |
| Site Log Output Format Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces strict JSON schema | None (AI sub-node) | Extract Site Log Fields | ## Extract log fields... |
| Build Log Row Data | n8n-nodes-base.code | Flattens AI output into spreadsheet row | Extract Site Log Fields | Append to Site Log Sheet | ## Save log and alert... |
| Append to Site Log Sheet | n8n-nodes-base.googleSheets | Appends row to Site log sheet tab | Build Log Row Data | Confirm Receipt to Crew, Check PM Attention Needed | ## Save log and alert... |
| Confirm Receipt to Crew | n8n-nodes-base.whatsApp | Confirms logging success to crew | Append to Site Log Sheet | None | ## Save log and alert... |
| Check PM Attention Needed | n8n-nodes-base.if | Checks if PM attention flag is set | Append to Site Log Sheet | Send PM Alert Email | ## Save log and alert... |
| Send PM Alert Email | n8n-nodes-base.gmail | Emails urgent alerts to PM | Check PM Attention Needed | None | ## Save log and alert... |
| Read Site Log for Today | n8n-nodes-base.googleSheets | Reads today's log rows | Route by Trigger Event | Read Projects for Reports | ## Collect report inputs... |
| Read Projects for Reports | n8n-nodes-base.googleSheets | Reads project report configuration | Read Site Log for Today | Group Updates by Project | ## Collect report inputs... |
| Group Updates by Project | n8n-nodes-base.code | Groups updates and calculates totals | Read Projects for Reports | Check for Todays Updates | ## Collect report inputs... |
| Check for Todays Updates | n8n-nodes-base.if | Branches by project activity status | Group Updates by Project | Compile Daily Report, Create No-Update Notice | ## Collect report inputs... |
| Compile Daily Report | @n8n/n8n-nodes-langchain.chainLlm | Generates daily project report via LLM | Check for Todays Updates | Construct Report Email | ## Compose report content... |
| OpenAI Model for Reports | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for reports (GPT-4o-mini) | None (AI sub-node) | Compile Daily Report | ## Compose report content... |
| Daily Report Format Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces daily report JSON schema | None (AI sub-node) | Compile Daily Report | ## Compose report content... |
| Construct Report Email | n8n-nodes-base.code | Constructs HTML email for updates | Compile Daily Report | Send Daily Report Email, Format Report Row | ## Send and archive reports... |
| Create No-Update Notice | n8n-nodes-base.code | Constructs fallback no-update email | Check for Todays Updates | Send Daily Report Email, Format Report Row | ## Send and archive reports... |
| Send Daily Report Email | n8n-nodes-base.gmail | Sends daily report/notice email | Construct Report Email, Create No-Update Notice | None | ## Send and archive reports... |
| Format Report Row | n8n-nodes-base.set | Formats report metadata for archive | Construct Report Email, Create No-Update Notice | Append to Daily Reports Sheet | ## Send and archive reports... |
| Append to Daily Reports Sheet | n8n-nodes-base.googleSheets | Appends summary to Daily reports tab | Format Report Row | None | ## Send and archive reports... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Triggers and Configuration
1. **`When WhatsApp Message Received`** (`n8n-nodes-base.whatsAppTrigger`): Set `updates` to `["messages"]`. Connect WhatsApp Business Cloud API credentials.
2. **`Every Evening at 6pm`** (`n8n-nodes-base.scheduleTrigger`): Set trigger interval to run daily at 18:00.
3. **`Set Configuration Parameters`** (`n8n-nodes-base.set`): Add string assignments for `companyName`, `sheetUrl`, `photosFolderId`, `officeEmail`, and `reportLanguage`.
4. **`Route by Trigger Event`** (`n8n-nodes-base.switch`): Configure rules to route based on whether `messages` exist or the schedule executed.

#### Step 2: Set Up Sender Resolution & Validation
5. **`Read Crew List from Sheets`** (`n8n-nodes-base.googleSheets`): Operation `read`, sheet name `Crew`, document ID from `sheetUrl`.
6. **`Read Project List from Sheets`** (`n8n-nodes-base.googleSheets`): Operation `read`, sheet name `Projects`, document ID from `sheetUrl`. Enable `executeOnce`.
7. **`Identify Sender and Project`** (`n8n-nodes-base.code`): Paste JavaScript code to parse contact metadata, match phone numbers against the crew tab, and extract `#ProjectName` tags.
8. **`Check if Registered Crew Member`** (`n8n-nodes-base.if`): Condition checking `{{ $json.registered }}` is true.
9. **`Reply: Not a Registered Number`** (`n8n-nodes-base.whatsApp`): Send text reply to unregistered numbers.

#### Step 3: Configure Media Processing Branches
10. **`Route by Message Type`** (`n8n-nodes-base.switch`): Match `audio`, `image`, `text`, with fallback to `Other`.
11. **Voice Note Path:**
    - `Fetch Voice Note URL`: WhatsApp node, resource `media`, operation `mediaUrlGet`.
    - `Download Voice Note`: HTTP Request node pointing to `{{ $json.url }}`, binary response, WhatsApp authentication.
    - `OpenAI Transcription`: OpenAI node, resource `audio`, operation `transcribe`.
12. **Photo Path:**
    - `Fetch Photo URL`: WhatsApp node, resource `media`, operation `mediaUrlGet`.
    - `Download Photo`: HTTP Request node, binary response.
    - `Describe Photo Using AI`: OpenAI node, resource `image`, operation `analyze`, model `gpt-4o-mini`.
    - `Save Photo to Google Drive`: Google Drive node, operation `upload`, folder ID from configuration. Set `onError` to `continueRegularOutput`.
    - `Merge Photo Descriptions`: Merge node, mode `combineByPosition`.
13. **`Normalize Update Into Text`** (`n8n-nodes-base.code`): Combines text, transcription, photo descriptions, and drive links.

#### Step 4: Configure LLM Extraction & Logging
14. **`Extract Site Log Fields`** (`@n8n/n8n-nodes-langchain.chainLlm`): Connect OpenAI Chat Model (`gpt-4o-mini`, temperature `0.1`) and Structured Output Parser.
15. **`Build Log Row Data`** (`n8n-nodes-base.code`): Flattens AI schema into a Google Sheets row object.
16. **`Append to Site Log Sheet`** (`n8n-nodes-base.googleSheets`): Operation `append`, sheet name `Site log`.
17. **`Confirm Receipt to Crew`** (`n8n-nodes-base.whatsApp`): Sends confirmation message.
18. **`Check PM Attention Needed`** (`n8n-nodes-base.if`): Checks if `Needs attention` equals `Yes`.
19. **`Send PM Alert Email`** (`n8n-nodes-base.gmail`): Sends HTML alert email via Gmail.

#### Step 5: Configure Evening Scheduled Reporting
20. **`Read Site Log for Today`** & **`Read Projects for Reports`** (`n8n-nodes-base.googleSheets`): Read today's site logs and project configurations.
21. **`Group Updates by Project`** (`n8n-nodes-base.code`): Aggregate totals and flags by project.
22. **`Check for Todays Updates`** (`n8n-nodes-base.if`): Branch based on `hasUpdates`.
23. **`Compile Daily Report`** (`@n8n/n8n-nodes-langchain.chainLlm`): Connect OpenAI Chat Model and Daily Report Structured Output Parser.
24. **`Construct Report Email`** / **`Create No-Update Notice`** (`n8n-nodes-base.code`): Build HTML emails.
25. **`Send Daily Report Email`** (`n8n-nodes-base.gmail`): Send daily report emails.
26. **`Format Report Row`** & **`Append to Daily Reports Sheet`** (`n8n-nodes-base.set` / `n8n-nodes-base.googleSheets`): Append summary rows to the `Daily reports` sheet.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Construction crews can report in any language; voice notes are transcribed and translated into the configured report language. | Multilingual support via OpenAI |
| Unregistered phone numbers are automatically rejected to prevent spam and unauthorized data entry. | Security and access control |
| AI extraction operates under strict constraints to prevent hallucination of quantities, dates, or progress metrics. | AI reliability guardrails |