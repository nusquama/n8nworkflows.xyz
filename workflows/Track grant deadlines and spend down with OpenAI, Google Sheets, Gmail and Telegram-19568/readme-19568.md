Track grant deadlines and spend down with OpenAI, Google Sheets, Gmail and Telegram

https://n8nworkflows.xyz/workflows/track-grant-deadlines-and-spend-down-with-openai--google-sheets--gmail-and-telegram-19568


# Track grant deadlines and spend down with OpenAI, Google Sheets, Gmail and Telegram

### 1. Workflow Overview

This workflow automates the tracking of grant awards, reporting deadlines, and financial drawdowns (spend-downs). It ingests incoming grant awards via webhook, leverages OpenAI to automatically convert complex award terms into structured reporting calendars with milestones and risk analysis, and updates a centralized Google Sheets tracker. It also monitors incoming spend transactions to calculate burn rates in real time, issues proactive alerts when burn rates or spend-by deadlines require attention, and performs scheduled weekly deadline sweeps and monthly portfolio digests.

The workflow logic is grouped into the following functional blocks:
- **1.1 Award Intake and Validation:** Receives raw grant award JSON data via webhook, normalizes the properties into standardized schema fields, and checks for data completeness and valid email addresses.
- **1.2 AI Reporting Calendar Generation:** Uses an OpenAI agent to parse award terms and generate a structured JSON reporting plan containing interim milestones, due dates, buffer days, and compliance risk notes.
- **1.3 Reporting Schedule Delivery & Registration:** Appends the master award record to Google Sheets, splits and logs individual reporting milestones into a dedicated deadlines tab, and notifies the grant lead via Gmail and Telegram.
- **1.4 Rejected Award Handling:** Catches malformed award payloads, formats an error notification email for administrators, alerts the team via Telegram, and logs the rejection details to an Errors sheet.
- **1.5 Spend Draw-Down Watch & Logging:** Ingests spend entries via webhook, validates required fields, reads the awards register from Google Sheets, computes updated cumulative spending and burn percentages, and logs the transaction.
- **1.6 Burn Alert & Award Update:** Triggers an AI-driven finance warning when budget consumption reaches or exceeds 80%, or when the spend-by deadline is within 45 days, distributing alerts via Gmail and Telegram while updating the master award row.
- **1.7 Rejected & Unknown Spend Handling:** Manages invalid spend payloads or drawdowns submitted against unlisted award IDs, routing them through administrative notification and error logging flows.
- **1.8 Weekly Deadline Sweep:** Runs every Monday morning via a schedule trigger to scan pending reporting milestones due within 30 days, draft AI-powered reminder emails with data checklists, send notifications, log the reminder, and update milestone statuses.
- **1.9 Monthly Portfolio Review:** Executes on the first of each month to evaluate all tracked awards for high burn rates or near-term expiration, generating an AI portfolio summary and emailing it to the finance lead.

---

### 2. Block-by-Block Analysis

#### 2.1 Award Intake and Validation
- **Overview:** Receives raw award payloads from external systems, standardizes key attributes, and validates mandatory fields and contact information before processing.
- **Nodes Involved:** `When Award Logged`, `Normalize award record`, `Validate award fields`
- **Node Details:**
  - **When Award Logged**
    - Type: `n8n-nodes-base.webhook`
    - Role: Webhook entry point listening for HTTP POST requests at path `grant-award`.
    - Connections: Output connects to `Normalize award record`.
    - Edge Cases: Unreachable endpoint or invalid JSON body payloads returning 4xx/5xx errors.
  - **Normalize award record**
    - Type: `n8n-nodes-base.set`
    - Role: Maps raw webhook body parameters into standardized data properties, auto-generating an `award_id` (`GR-[timestamp]`) if missing, trimming string fields, and casting monetary and numerical thresholds.
    - Expressions: Uses JavaScript ternary operations to assign fallback IDs, clean whitespace, and inject ISO timestamps (`$now.toISO()`).
    - Connections: Input from `When Award Logged`; output connects to `Validate award fields`.
  - **Validate award fields**
    - Type: `n8n-nodes-base.if`
    - Role: Evaluates data integrity by checking that funder name, program name, amount (> 0), end date, and valid email formats (`@`) for both program lead and funder contact are present.
    - Connections: Input from `Normalize award record`; true branch connects to `AI plan reporting calendar`, false branch connects to `Build rejected award alert`.

---

#### 2.2 AI Reporting Calendar Generation
- **Overview:** Processes validated award specifications through an OpenAI agent configured to output a strict JSON reporting calendar and risk assessment.
- **Nodes Involved:** `AI plan reporting calendar`, `Grant calendar model`, `Build reporting plan`
- **Node Details:**
  - **AI plan reporting calendar**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Role: LangChain agent utilizing a system prompt to act as a nonprofit compliance planner, transforming raw award terms into structured milestone schedules.
    - Expressions: Combines award metadata properties into a single prompt string.
    - Connections: Input from `Validate award fields` (true branch); connected to `Grant calendar model` via AI language model input; output connects to `Build reporting plan`.
  - **Grant calendar model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - Role: OpenAI chat model provider configured with `gpt-5.6-luna`.
    - Connections: Connected to `AI plan reporting calendar` via `ai_languageModel`.
  - **Build reporting plan**
    - Type: `n8n-nodes-base.set`
    - Role: Raw JavaScript execution node that parses the JSON output string from the AI agent, cleans and validates milestone arrays, and bundles master award metrics.
    - Connections: Input from `AI plan reporting calendar`; outputs connect to both `Append award to tracker` and `Split reporting milestones`.

---

#### 2.3 Reporting Schedule Delivery & Registration
- **Overview:** Persists the validated award and its individual deadlines into Google Sheets, then dispatches confirmation notifications via Gmail and Telegram.
- **Nodes Involved:** `Append award to tracker`, `Email reporting calendar to grant lead`, `Alert new award on Telegram`, `Split reporting milestones`, `Build deadline row`, `Append deadline rows`
- **Node Details:**
  - **Append award to tracker**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Appends a new row to the `Awards` sheet tab containing financial and administrative award details.
    - Connections: Input from `Build reporting plan`; output connects to `Email reporting calendar to grant lead`.
  - **Email reporting calendar to grant lead**
    - Type: `n8n-nodes-base.gmail`
    - Role: Sends an HTML-formatted email to the program lead detailing the newly tracked award, its reporting schedule, buffer intervals, and compliance risks.
    - Expressions: Dynamically references `program_lead` and maps milestone array attributes into HTML rows.
    - Connections: Input from `Append award to tracker`; output connects to `Alert new award on Telegram`.
  - **Alert new award on Telegram**
    - Type: `n8n-nodes-base.telegram`
    - Role: Broadcasts a text summary of the new grant award to a designated Telegram chat channel.
    - Connections: Input from `Email reporting calendar to grant lead`.
  - **Split reporting milestones**
    - Type: `n8n-nodes-base.splitOut`
    - Role: Deconstructs the `milestones` JSON array from the reporting plan object into individual items for row-by-row insertion.
    - Connections: Input from `Build reporting plan`; output connects to `Build deadline row`.
  - **Build deadline row**
    - Type: `n8n-nodes-base.set`
    - Role: Generates unique milestone IDs, computes buffer dates by subtracting buffer days using Luxon datetime operations, calculates days remaining until due dates, and sets initial status to `pending`.
    - Connections: Input from `Split reporting milestones`; output connects to `Append deadline rows`.
  - **Append deadline rows**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Appends each generated milestone row to the `Deadlines` sheet tab.
    - Connections: Input from `Build deadline row`.

---

#### 2.4 Rejected Award Handling
- **Overview:** Captures invalid award submissions, constructs descriptive error logs, notifies administrators via email, posts alerts to Telegram, and records the failure in Google Sheets.
- **Nodes Involved:** `Build rejected award alert`, `Email admin about rejected award`, `Alert rejected award on Telegram`, `Log award error`
- **Node Details:**
  - **Build rejected award alert**
    - Type: `n8n-nodes-base.set`
    - Role: Evaluates which specific validation checks failed and constructs structured error identifiers, email bodies, and Telegram notification texts.
    - Connections: Input from `Validate award fields` (false branch); output connects to `Email admin about rejected award`.
  - **Email admin about rejected award**
    - Type: `n8n-nodes-base.gmail`
    - Role: Emails the administrative recipient (`user@example.com`) regarding the rejected submission.
    - Connections: Input from `Build rejected award alert`; output connects to `Alert rejected award on Telegram`.
  - **Alert rejected award on Telegram**
    - Type: `n8n-nodes-base.telegram`
    - Role: Posts rejection warnings to the team's Telegram channel.
    - Connections: Input from `Email admin about rejected award`; output connects to `Log award error`.
  - **Log award error**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Appends error metadata, reasons, and details to the `Errors` sheet tab.
    - Connections: Input from `Alert rejected award on Telegram`.

---

#### 2.5 Spend Draw-Down Watch & Logging
- **Overview:** Receives spend transaction webhooks, validates parameters, queries the master awards database, computes cumulative spend totals and burn rates, and records the spend entry.
- **Nodes Involved:** `When Grant Spend Logged`, `Normalize spend entry`, `Validate spend entry`, `Read awards for spend watch`, `Collapse award rows`, `Compute burn rate`, `Award found?`, `Log spend entry`
- **Node Details:**
  - **When Grant Spend Logged**
    - Type: `n8n-nodes-base.webhook`
    - Role: Webhook entry point listening for HTTP POST requests at path `grant-spend`.
    - Connections: Output connects to `Normalize spend entry`.
  - **Normalize spend entry**
    - Type: `n8n-nodes-base.set`
    - Role: Standardizes incoming spend webhook payloads, assigning default spend IDs (`SP-[timestamp]`) and timestamps.
    - Connections: Input from `When Grant Spend Logged`; output connects to `Validate spend entry`.
  - **Validate spend entry**
    - Type: `n8n-nodes-base.if`
    - Role: Confirms the presence of a target `award_id`, a spend amount greater than zero, and a specified category.
    - Connections: Input from `Normalize spend entry`; true branch connects to `Read awards for spend watch`, false branch connects to `Build rejected spend alert`.
  - **Read awards for spend watch**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Fetches all records from the `Awards` sheet tab to allow matching against the incoming spend transaction.
    - Connections: Input from `Validate spend entry` (true branch); output connects to `Collapse award rows`.
  - **Collapse award rows**
    - Type: `n8n-nodes-base.limit`
    - Role: Consolidates retrieved sheet rows into a unified data array context for downstream JavaScript mapping operations.
    - Connections: Input from `Read awards for spend watch`; output connects to `Compute burn rate`.
  - **Compute burn rate**
    - Type: `n8n-nodes-base.set`
    - Role: Executes JavaScript logic to locate the matching award record, aggregate prior spending with current spend, calculate updated burn percentages, and determine days remaining until the spend-by date.
    - Connections: Input from `Collapse award rows`; output connects to `Award found?`.
  - **Award found?**
    - Type: `n8n-nodes-base.if`
    - Role: Evaluates whether the award ID exists in the tracked register (`award_found` boolean).
    - Connections: Input from `Compute burn rate`; true branch connects to `Log spend entry`, false branch connects to `Build unknown award alert`.
  - **Log spend entry**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Appends the validated spend transaction to the `SpendLog` sheet tab.
    - Connections: Input from `Award found?` (true branch); output connects to `Update award spend total`.

---

#### 2.6 Burn Alert & Award Update
- **Overview:** Updates the master award's spent total in Google Sheets, checks burn and expiration thresholds, and invokes an AI finance reviewer to generate warning notes if criteria are met.
- **Nodes Involved:** `Update award spend total`, `Burn threshold reached?`, `AI write spend note`, `Spend note model`, `Parse spend note`, `Email finance about burn rate`, `Alert burn rate on Telegram`
- **Node Details:**
  - **Update award spend total**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Updates the `spent_to_date` value for the specific `award_id` in the `Awards` sheet tab.
    - Connections: Input from `Log spend entry`; output connects to `Burn threshold reached?`.
  - **Burn threshold reached?**
    - Type: `n8n-nodes-base.if`
    - Role: Checks if budget burn percentage is $\ge 80\%$ OR days remaining until spend-by date is $\le 45$ days.
    - Connections: Input from `Update award spend total`; true branch connects to `AI write spend note`.
  - **AI write spend note**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Role: Acting as a grants finance reviewer, writes a short internal finance memo and chat alerts based on consumption thresholds and restriction categories.
    - Connections: Input from `Burn threshold reached?` (true branch); connected to `Spend note model` via AI language model input; output connects to `Parse spend note`.
  - **Spend note model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - Role: OpenAI model provider configured with `gpt-5.6-terra`.
    - Connections: Connected to `AI write spend note` via `ai_languageModel`.
  - **Parse spend note**
    - Type: `n8n-nodes-base.set`
    - Role: Parses the JSON output from the spend note AI agent and structures email subjects, bodies, and alert payloads.
    - Connections: Input from `AI write spend note`; output connects to `Email finance about burn rate`.
  - **Email finance about burn rate**
    - Type: `n8n-nodes-base.gmail`
    - Role: Sends the financial advisory note to the program/grant lead.
    - Connections: Input from `Parse spend note`; output connects to `Alert burn rate on Telegram`.
  - **Alert burn rate on Telegram**
    - Type: `n8n-nodes-base.telegram`
    - Role: Posts the burn rate warning text to the designated Telegram chat.
    - Connections: Input from `Email finance about burn rate`.

---

#### 2.7 Rejected & Unknown Spend Handling
- **Overview:** Manages invalid spend payloads or spend requests submitted for unrecorded grant IDs, logging errors and alerting team members.
- **Nodes Involved:** `Build rejected spend alert`, `Email admin about rejected spend`, `Alert rejected spend on Telegram`, `Log spend error`, `Build unknown award alert`, `Alert unknown award on Telegram`, `Log unknown award error`
- **Node Details:**
  - **Build rejected spend alert**
    - Type: `n8n-nodes-base.set`
    - Role: Formats validation error notices for rejected spend entries.
    - Connections: Input from `Validate spend entry` (false branch); output connects to `Email admin about rejected spend`.
  - **Email admin about rejected spend**
    - Type: `n8n-nodes-base.gmail`
    - Role: Emails administrators regarding invalid spend submission payloads.
    - Connections: Input from `Build rejected spend alert`; output connects to `Alert rejected spend on Telegram`.
  - **Alert rejected spend on Telegram**
    - Type: `n8n-nodes-base.telegram`
    - Role: Broadcasts spend rejection notices to Telegram.
    - Connections: Input from `Email admin about rejected spend`; output connects to `Log spend error`.
  - **Log spend error**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Appends spend validation error records to the `Errors` sheet tab.
    - Connections: Input from `Alert rejected spend on Telegram`.
  - **Build unknown award alert**
    - Type: `n8n-nodes-base.set`
    - Role: Formats error messages when spend transactions reference an `award_id` absent from the register.
    - Connections: Input from `Award found?` (false branch); output connects to `Alert unknown award on Telegram`.
  - **Alert unknown award on Telegram**
    - Type: `n8n-nodes-base.telegram`
    - Role: Alerts the team via Telegram about unregistered award spend attempts.
    - Connections: Input from `Build unknown award alert`; output connects to `Log unknown award error`.
  - **Log unknown award error**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Appends unregistered award error details to the `Errors` sheet tab.
    - Connections: Input from `Alert unknown award on Telegram`.

---

#### 2.8 Weekly Deadline Sweep
- **Overview:** Runs every Monday to scan pending reporting milestones due within 30 days, generates AI reminder drafts with data checklists, emails leads, logs reminders, and marks milestones as sent.
- **Nodes Involved:** `Weekly deadline sweep`, `Read deadlines for sweep`, `Compute deadline risk`, `Reminder window?`, `AI write deadline reminder`, `Deadline reminder model`, `Parse reminder draft`, `Email report reminder to grant lead`, `Alert deadline on Telegram`, `Log deadline reminder`, `Mark deadline reminder sent`
- **Node Details:**
  - **Weekly deadline sweep**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Cron schedule trigger executing every Monday at 08:00 (`0 8 * * 1`).
    - Connections: Output connects to `Read deadlines for sweep`.
  - **Read deadlines for sweep**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Retrieves all entries from the `Deadlines` sheet tab.
    - Connections: Input from `Weekly deadline sweep`; output connects to `Compute deadline risk`.
  - **Compute deadline risk**
    - Type: `n8n-nodes-base.set`
    - Role: Calculates days remaining until due dates and categorizes reminder urgency levels.
    - Connections: Input from `Read deadlines for sweep`; output connects to `Reminder window?`.
  - **Reminder window?**
    - Type: `n8n-nodes-base.if`
    - Role: Filters deadlines where days left is $\le 30$ days AND row status equals `pending`.
    - Connections: Input from `Compute deadline risk`; true branch connects to `AI write deadline reminder`.
  - **AI write deadline reminder**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Role: Acts as a compliance assistant to draft customized reminder emails and data checklists.
    - Connections: Input from `Reminder window?` (true branch); connected to `Deadline reminder model` via AI language model input; output connects to `Parse reminder draft`.
  - **Deadline reminder model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - Role: OpenAI model provider configured with `gpt-4o-mini`.
    - Connections: Connected to `AI write deadline reminder` via `ai_languageModel`.
  - **Parse reminder draft**
    - Type: `n8n-nodes-base.set`
    - Role: Parses agent output JSON and constructs HTML email bodies incorporating data checklists and review dates.
    - Connections: Input from `AI write deadline reminder`; output connects to `Email report reminder to grant lead`.
  - **Email report reminder to grant lead**
    - Type: `n8n-nodes-base.gmail`
    - Role: Sends the deadline reminder email to the deadline owner/grant lead.
    - Connections: Input from `Parse reminder draft`; output connects to `Alert deadline on Telegram`.
  - **Alert deadline on Telegram**
    - Type: `n8n-nodes-base.telegram`
    - Role: Posts deadline reminder notices to the Telegram channel.
    - Connections: Input from `Email report reminder to grant lead`; output connects to `Log deadline reminder`.
  - **Log deadline reminder**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Appends reminder transmission records to the `ReminderLog` sheet tab.
    - Connections: Input from `Alert deadline on Telegram`; output connects to `Mark deadline reminder sent`.
  - **Mark deadline reminder sent**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Updates the status column of the specific milestone row in the `Deadlines` sheet tab to `reminder sent`.
    - Connections: Input from `Log deadline reminder`.

---

#### 2.9 Monthly Portfolio Review
- **Overview:** Executes on the first of every month to evaluate the entire awards portfolio, compiling summaries for grants requiring financial attention and emailing a consolidated AI digest.
- **Nodes Involved:** `Monthly spend down review`, `Read awards for review`, `Combine award portfolio`, `Compute portfolio position`, `Portfolio needs attention?`, `AI write portfolio digest`, `Portfolio digest model`, `Parse portfolio digest`, `Email portfolio digest to finance lead`
- **Node Details:**
  - **Monthly spend down review**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Cron schedule trigger executing on the 1st of every month at 09:00 (`0 9 1 * *`).
    - Connections: Output connects to `Read awards for review`.
  - **Read awards for review**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Fetches all records from the `Awards` sheet tab.
    - Connections: Input from `Monthly spend down review`; output connects to `Combine award portfolio`.
  - **Combine award portfolio**
    - Type: `n8n-nodes-base.aggregate`
    - Role: Aggregates all item data rows into a single combined collection array.
    - Connections: Input from `Read awards for review`; output connects to `Compute portfolio position`.
  - **Compute portfolio position**
    - Type: `n8n-nodes-base.set`
    - Role: Computes burn percentages and days to spend-by for all portfolio awards, filtering items requiring attention ($\text{burn} \ge 80\%$ or $\text{days} \le 45$).
    - Connections: Input from `Combine award portfolio`; output connects to `Portfolio needs attention?`.
  - **Portfolio needs attention?**
    - Type: `n8n-nodes-base.if`
    - Role: Checks if the count of awards needing attention is $\ge 1$.
    - Connections: Input from `Compute portfolio position`; true branch connects to `AI write portfolio digest`.
  - **AI write portfolio digest**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Role: Acts as an assistant to the nonprofit finance lead, synthesizing portfolio data into an executive summary.
    - Connections: Input from `Portfolio needs attention?` (true branch); connected to `Portfolio digest model` via AI language model input; output connects to `Parse portfolio digest`.
  - **Portfolio digest model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - Role: OpenAI model provider configured with `gpt-5.6-terra`.
    - Connections: Connected to `AI write portfolio digest` via `ai_languageModel`.
  - **Parse portfolio digest**
    - Type: `n8n-nodes-base.set`
    - Role: Parses the JSON output from the portfolio digest agent and constructs email subject and body payloads.
    - Connections: Input from `AI write portfolio digest`; output connects to `Email portfolio digest to finance lead`.
  - **Email portfolio digest to finance lead**
    - Type: `n8n-nodes-base.gmail`
    - Role: Emails the monthly portfolio review digest to the finance lead (`user@example.com`).
    - Connections: Input from `Parse portfolio digest`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Master workflow documentation and instructions | None | None | ## 09-12-01 track grant reporting deadlines and spend down with AI... |
| Sticky Note: Award intake and validation | n8n-nodes-base.stickyNote | Visual container for award intake notes | None | None | ## Award intake and validation... |
| Sticky Note: AI reporting calendar | n8n-nodes-base.stickyNote | Visual container for AI reporting calendar notes | None | None | ## AI reporting calendar... |
| Sticky Note: Reporting schedule delivery | n8n-nodes-base.stickyNote | Visual container for schedule delivery notes | None | None | ## Reporting schedule delivery... |
| Sticky Note: Rejected award alerts | n8n-nodes-base.stickyNote | Visual container for rejected award alerts notes | None | None | ## Rejected award alerts... |
| Sticky Note: Spend draw down watch | n8n-nodes-base.stickyNote | Visual container for spend draw down notes | None | None | ## Spend draw down watch... |
| Sticky Note: Burn alert and award update | n8n-nodes-base.stickyNote | Visual container for burn alert notes | None | None | ## Burn alert and award update... |
| Sticky Note: Spend entry rejected | n8n-nodes-base.stickyNote | Visual container for rejected spend entry notes | None | None | ## Spend entry rejected... |
| Sticky Note: Unknown award | n8n-nodes-base.stickyNote | Visual container for unknown award notes | None | None | ## Unknown award... |
| Sticky Note: Weekly deadline sweep | n8n-nodes-base.stickyNote | Visual container for weekly sweep notes | None | None | ## Weekly deadline sweep... |
| Sticky Note: Monthly portfolio review | n8n-nodes-base.stickyNote | Visual container for monthly review notes | None | None | ## Monthly portfolio review... |
| When Award Logged | n8n-nodes-base.webhook | Receives incoming grant award POST requests | None | Normalize award record | Award intake and validation |
| Normalize award record | n8n-nodes-base.set | Standardizes award properties and assigns fallback IDs | When Award Logged | Validate award fields | Award intake and validation |
| Validate award fields | n8n-nodes-base.if | Validates required award fields and contact email formats | Normalize award record | AI plan reporting calendar, Build rejected award alert | Award intake and validation |
| AI plan reporting calendar | @n8n/n8n-nodes-langchain.agent | Generates JSON reporting milestones and compliance risks | Validate award fields | Build reporting plan | AI reporting calendar |
| Grant calendar model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides OpenAI gpt-5.6-luna model for calendar generation | None | AI plan reporting calendar | AI reporting calendar |
| Build reporting plan | n8n-nodes-base.set | Parses AI agent output and structures award/milestone data | AI plan reporting calendar | Append award to tracker, Split reporting milestones | AI reporting calendar |
| Append award to tracker | n8n-nodes-base.googleSheets | Appends award record to Awards sheet tab | Build reporting plan | Email reporting calendar to grant lead | Reporting schedule delivery |
| Email reporting calendar to grant lead | n8n-nodes-base.gmail | Emails reporting calendar to the grant lead | Append award to tracker | Alert new award on Telegram | Reporting schedule delivery |
| Alert new award on Telegram | n8n-nodes-base.telegram | Sends new award alert message to Telegram channel | Email reporting calendar to grant lead | None | Reporting schedule delivery |
| Split reporting milestones | n8n-nodes-base.splitOut | Splits milestones array into individual items | Build reporting plan | Build deadline row | Reporting schedule delivery |
| Build deadline row | n8n-nodes-base.set | Calculates buffer dates and days left for milestones | Split reporting milestones | Append deadline rows | Reporting schedule delivery |
| Append deadline rows | n8n-nodes-base.googleSheets | Appends individual deadline rows to Deadlines sheet tab | Build deadline row | None | Reporting schedule delivery |
| Build rejected award alert | n8n-nodes-base.set | Formats error details and notification texts for rejected awards | Validate award fields | Email admin about rejected award | Rejected award alerts |
| Email admin about rejected award | n8n-nodes-base.gmail | Emails admin about rejected award submission | Build rejected award alert | Alert rejected award on Telegram | Rejected award alerts |
| Alert rejected award on Telegram | n8n-nodes-base.telegram | Sends award rejection alert to Telegram channel | Email admin about rejected award | Log award error | Rejected award alerts |
| Log award error | n8n-nodes-base.googleSheets | Appends award intake error record to Errors sheet tab | Alert rejected award on Telegram | None | Rejected award alerts |
| When Grant Spend Logged | n8n-nodes-base.webhook | Receives grant spend POST requests | None | Normalize spend entry | Spend draw down watch |
| Normalize spend entry | n8n-nodes-base.set | Standardizes incoming spend entry payload | When Grant Spend Logged | Validate spend entry | Spend draw down watch |
| Validate spend entry | n8n-nodes-base.if | Validates mandatory spend fields and amount > 0 | Normalize spend entry | Read awards for spend watch, Build rejected spend alert | Spend draw down watch |
| Read awards for spend watch | n8n-nodes-base.googleSheets | Reads awards sheet to match incoming spend transactions | Validate spend entry | Collapse award rows | Spend draw down watch |
| Collapse award rows | n8n-nodes-base.limit | Collapses sheet rows into array context | Read awards for spend watch | Compute burn rate | Spend draw down watch |
| Compute burn rate | n8n-nodes-base.set | Calculates updated spend totals, burn percentages, and days left | Collapse award rows | Award found? | Spend draw down watch |
| Award found? | n8n-nodes-base.if | Verifies if matching award exists in awards register | Compute burn rate | Log spend entry, Build unknown award alert | Spend draw down watch |
| Log spend entry | n8n-nodes-base.googleSheets | Appends spend transaction record to SpendLog sheet tab | Award found? | Update award spend total | Spend draw down watch |
| Burn threshold reached? | n8n-nodes-base.if | Checks if burn $\ge 80\%$ or days to spend-by $\le 45$ | Update award spend total | AI write spend note | Burn alert and award update |
| AI write spend note | @n8n/n8n-nodes-langchain.agent | Generates internal finance warning memo and risk level | Burn threshold reached? | Parse spend note | Burn alert and award update |
| Spend note model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides OpenAI gpt-5.6-terra model for spend notes | None | AI write spend note | Burn alert and award update |
| Parse spend note | n8n-nodes-base.set | Parses spend note agent JSON output | AI write spend note | Email finance about burn rate | Burn alert and award update |
| Email finance about burn rate | n8n-nodes-base.gmail | Emails finance advisory note to program lead | Parse spend note | Alert burn rate on Telegram | Burn alert and award update |
| Alert burn rate on Telegram | n8n-nodes-base.telegram | Sends burn rate warning message to Telegram | Email finance about burn rate | None | Burn alert and award update |
| Update award spend total | n8n-nodes-base.googleSheets | Updates spent_to_date value in Awards sheet tab | Log spend entry | Burn threshold reached? | Burn alert and award update |
| Build rejected spend alert | n8n-nodes-base.set | Formats error details for invalid spend payloads | Validate spend entry | Email admin about rejected spend | Spend entry rejected |
| Email admin about rejected spend | n8n-nodes-base.gmail | Emails admin about rejected spend entry | Build rejected spend alert | Alert rejected spend on Telegram | Spend entry rejected |
| Alert rejected spend on Telegram | n8n-nodes-base.telegram | Sends spend rejection alert to Telegram | Email admin about rejected spend | Log spend error | Spend entry rejected |
| Log spend error | n8n-nodes-base.googleSheets | Appends spend error record to Errors sheet tab | Alert rejected spend on Telegram | None | Spend entry rejected |
| Build unknown award alert | n8n-nodes-base.set | Formats error notice for spend against unknown award ID | Award found? | Alert unknown award on Telegram | Unknown award |
| Alert unknown award on Telegram | n8n-nodes-base.telegram | Sends unknown award warning to Telegram | Build unknown award alert | Log unknown award error | Unknown award |
| Log unknown award error | n8n-nodes-base.googleSheets | Appends unknown award error record to Errors sheet tab | Alert unknown award on Telegram | None | Unknown award |
| Weekly deadline sweep | n8n-nodes-base.scheduleTrigger | Cron trigger running every Monday at 08:00 | None | Read deadlines for sweep | Weekly deadline sweep |
| Read deadlines for sweep | n8n-nodes-base.googleSheets | Reads all rows from Deadlines sheet tab | Weekly deadline sweep | Compute deadline risk | Weekly deadline sweep |
| Compute deadline risk | n8n-nodes-base.set | Calculates days left and reminder urgency levels | Read deadlines for sweep | Reminder window? | Weekly deadline sweep |
| Reminder window? | n8n-nodes-base.if | Filters pending deadlines due within 30 days | Compute deadline risk | AI write deadline reminder | Weekly deadline sweep |
| AI write deadline reminder | @n8n/n8n-nodes-langchain.agent | Generates reminder email draft and data checklist | Reminder window? | Parse reminder draft | Weekly deadline sweep |
| Deadline reminder model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides OpenAI gpt-4o-mini model for deadline reminders | None | AI write deadline reminder | Weekly deadline sweep |
| Parse reminder draft | n8n-nodes-base.set | Parses reminder agent output JSON and formats HTML email | AI write deadline reminder | Email report reminder to grant lead | Weekly deadline sweep |
| Email report reminder to grant lead | n8n-nodes-base.gmail | Sends deadline reminder email to grant lead | Parse reminder draft | Alert deadline on Telegram | Weekly deadline sweep |
| Alert deadline on Telegram | n8n-nodes-base.telegram | Sends deadline reminder notice to Telegram | Email report reminder to grant lead | Log deadline reminder | Weekly deadline sweep |
| Log deadline reminder | n8n-nodes-base.googleSheets | Appends reminder transmission record to ReminderLog sheet | Alert deadline on Telegram | Mark deadline reminder sent | Weekly deadline sweep |
| Mark deadline reminder sent | n8n-nodes-base.googleSheets | Updates deadline status to reminder sent in Deadlines sheet | Log deadline reminder | None | Weekly deadline sweep |
| Monthly spend down review | n8n-nodes-base.scheduleTrigger | Cron trigger running on the 1st of every month at 09:00 | None | Read awards for review | Monthly portfolio review |
| Read awards for review | n8n-nodes-base.googleSheets | Reads all records from Awards sheet tab | Monthly spend down review | Combine award portfolio | Monthly portfolio review |
| Combine award portfolio | n8n-nodes-base.aggregate | Aggregates all award rows into array collection | Read awards for review | Compute portfolio position | Monthly portfolio review |
| Compute portfolio position | n8n-nodes-base.set | Evaluates portfolio items and filters those needing attention | Combine award portfolio | Portfolio needs attention? | Monthly portfolio review |
| Portfolio needs attention? | n8n-nodes-base.if | Checks if attention count is $\ge 1$ | Compute portfolio position | AI write portfolio digest | Monthly portfolio review |
| AI write portfolio digest | @n8n/n8n-nodes-langchain.agent | Synthesizes portfolio status into executive summary digest | Portfolio needs attention? | Parse portfolio digest | Monthly portfolio review |
| Portfolio digest model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides OpenAI gpt-5.6-terra model for portfolio digest | None | AI write portfolio digest | Monthly portfolio review |
| Parse portfolio digest | n8n-nodes-base.set | Parses digest agent JSON output and formats email subject/body | AI write portfolio digest | Email portfolio digest to finance lead | Monthly portfolio review |
| Email portfolio digest to finance lead | n8n-nodes-base.gmail | Emails portfolio review digest to finance lead | Parse portfolio digest | None | Monthly portfolio review |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow in n8n manually. Ensure you have configured Google Sheets with five tabs named **Awards**, **Deadlines**, **SpendLog**, **ReminderLog**, and **Errors**, using spreadsheet ID `17grTWLZ7D6EfsNw9QRKOfqyQkojmRbNSKhokYT7SWK0` (or update node parameters to match your custom document ID).

#### Step 1: Award Intake and Validation
1. Create a **Webhook** node (`When Award Logged`). Set HTTP Method to `POST` and Path to `grant-award`.
2. Create a **Set** node (`Normalize award record`). Connect the Webhook output to it. Configure assignments to map incoming body parameters (`award_id`, `funder`, `program`, `amount`, `start_date`, `end_date`, `report_cadence`, `report_due_days`, `spend_by`, `program_lead`, `funder_contact`, `restricted_categories`, and `received_at`).
3. Create an **If** node (`Validate award fields`). Connect `Normalize award record` to it. Configure conditions to ensure `funder` and `program` are not empty, `amount > 0`, `end_date` is not empty, and both `program_lead` and `funder_contact` contain `@`.

#### Step 2: AI Reporting Calendar Generation
4. Create an **Advanced AI Agent** node (`AI plan reporting calendar`). Connect the `true` branch of `Validate award fields` to its main input. Set prompt type to define and provide a system message instructing the agent to return strict JSON with `milestones`, `buffer_days`, `risk_note`, and `first_action`.
5. Create an **OpenAI Chat Model** node (`Grant calendar model`) using model `gpt-5.6-luna` with appropriate OpenAI credentials. Connect it to the agent's `ai_languageModel` input.
6. Create a **Set** node (`Build reporting plan`) using raw mode. Connect the agent's output to parse the JSON string, sanitize milestone records, and bundle master award properties.

#### Step 3: Reporting Schedule Delivery & Registration
7. Create a **Google Sheets** node (`Append award to tracker`). Connect `Build reporting plan` to it. Set operation to `Append`, document ID, and sheet name to `Awards`, mapping all award parameters.
8. Create a **Gmail** node (`Email reporting calendar-to-` / `Email reporting calendar to grant lead`). Connect `Append award to tracker` to it. Set recipient to `{{ $('Build reporting plan').first().json.program_lead }}` and configure the HTML message template using milestone arrays. Connect Gmail credentials.
9. Create a **Telegram** node (`Alert new award on Telegram`). Connect the Gmail node output to it. Configure chat ID (`123456789`) and text expression for the new award alert. Connect Telegram credentials.
10. Create a **Split Out** node (`Split reporting milestones`). Connect `Build reporting plan` to it, setting field to split out as `milestones`.
11. Create a **Set** node (`Build deadline row`). Connect `Split reporting milestones` to it to generate `deadline_id`, calculate buffer dates using Luxon, compute days left, and set status to `pending`.
12. Create a **Google Sheets** node (`Append deadline rows`). Connect `Build deadline row` to it. Set operation to `Append`, document ID, and sheet name to `Deadlines`, mapping deadline attributes.

#### Step 4: Rejected Award Handling
13. Create a **Set** node (`Build rejected award alert`). Connect the `false` branch of `Validate award fields` to it to formulate error reasons, admin email bodies, and Telegram warning text.
14. Create a **Gmail** node (`Email admin about rejected award`). Connect it to send error notifications to `user@example.com`.
15. Create a **Telegram** node (`Alert rejected award on Telegram`). Connect it to post rejection warnings to the team channel.
16. Create a **Google Sheets** node (`Log award error`). Connect it to append error details to the `Errors` sheet tab.

#### Step 5: Spend Draw-Down Watch & Logging
17. Create a **Webhook** node (`When Grant Spend Logged`). Set HTTP Method to `POST` and Path to `grant-spend`.
18. Create a **Set** node (`Normalize spend entry`). Connect the webhook to standardize `spend_id`, `award_id`, `amount_spent`, `category`, `spent_on`, and `received_at`.
19. Create an **If** node (`Validate spend entry`). Check that `award_id` is not empty, `amount_spent > 0`, and `category` is not empty.
20. Create a **Google Sheets** node (`Read awards for spend watch`). Connect the `true` branch to read all rows from the `Awards` sheet tab.
21. Create a **Limit** node (`Collapse award rows`). Connect it to collapse retrieved sheet rows.
22. Create a **Set** node (`Compute burn rate`). Connect it to calculate cumulative spend, burn percentage (`burn_pct`), and days to spend-by.
23. Create an **If** node (`Award found?`). Check that `award_found` is true.
24. Create a **Google Sheets** node (`Log spend entry`). Connect the `true` branch to append the transaction to the `SpendLog` sheet tab.

#### Step 6: Burn Alert & Award Update
25. Create a **Google Sheets** node (`Update award spend total`). Connect `Log spend entry` to it. Set operation to `Update`, match on `award_id`, and update `spent_to_date` and `status`.
26. Create an **If** node (`Burn threshold reached?`). Connect `Update award spend total` to it. Set conditions: `burn_pct >= 80` OR `days_to_spend_by <= 45`.
27. Create an **Advanced AI Agent** node (`AI write spend note`). Connect the `true` branch to it with a system prompt instructing the model to output strict JSON keys (`note_subject`, `note_body`, `discord_alert`, `risk_level`).
28. Create an **OpenAI Chat Model** node (`Spend note model`) using model `gpt-5.6-terra`. Connect it to the agent's `ai_languageModel` input.
29. Create a **Set** node (`Parse spend note`) in raw mode to extract JSON fields.
30. Create a **Gmail** node (`Email finance about burn rate`). Connect it to email the warning note to `program_lead`.
31. Create a **Telegram** node (`Alert burn rate on Telegram`). Connect it to post the alert text to Telegram.

#### Step 7: Rejected & Unknown Spend Handling
32. Configure the `false` branch of `Validate spend entry` to feed into **Build rejected spend alert**, **Email admin about rejected spend**, **Alert rejected spend on Telegram**, and **Log spend error** (mirroring Step 4).
33. Configure the `false` branch of `Award found?` to feed into **Build unknown award alert**, **Alert unknown award on Telegram**, and **Log unknown award error** (appending to the `Errors` tab).

#### Step 8: Weekly Deadline Sweep
34. Create a **Schedule Trigger** node (`Weekly deadline sweep`). Set interval cron expression to `0 8 * * 1` (Monday at 08:00).
35. Create a **Google Sheets** node (`Read deadlines for sweep`). Connect it to read all rows from the `Deadlines` sheet tab.
36. Create a **Set** node (`Compute deadline risk`) to calculate days left and `reminder_level`.
37. Create an **If** node (`Reminder window?`). Check `days_left <= 30` AND `row_status == 'pending'`.
38. Create an **Advanced AI Agent** node (`AI write deadline reminder`) paired with an **OpenAI Chat Model** (`Deadline reminder model`) using `gpt-4o-mini` to generate reminder emails and data checklists.
39. Create a **Set** node (`Parse reminder draft`) to format HTML email bodies and alert texts.
40. Create a **Gmail** node (`Email report reminder to grant lead`) to send the reminder.
41. Create a **Telegram** node (`Alert deadline on Telegram`) to broadcast the reminder.
42. Create a **Google Sheets** node (`Log deadline reminder`) to append records to `ReminderLog`.
43. Create a **Google Sheets** node (`Mark deadline reminder sent`) to update milestone status to `reminder sent` in `Deadlines` matching on `deadline_id`.

#### Step 9: Monthly Portfolio Review
44. Create a **Schedule Trigger** node (`Monthly spend down review`). Set cron expression to `0 9 1 * *` (1st of every month at 09:00).
45. Create a **Google Sheets** node (`Read awards for review`) to read all records from the `Awards` sheet tab.
46. Create an **Aggregate** node (`Combine award portfolio`) to aggregate item data.
47. Create a **Set** node (`Compute portfolio position`) to compute portfolio metrics and filter items needing attention.
48. Create an **If** node (`Portfolio needs attention?`). Check `attention_count >= 1`.
49. Create an **Advanced AI Agent** node (`AI write portfolio digest`) paired with an **OpenAI Chat Model** (`Portfolio digest model`) using `gpt-5.6-terra` to generate executive portfolio summaries.
50. Create a **Set** node (`Parse portfolio digest`) to format digest emails.
51. Create a **Gmail** node (`Email portfolio digest to finance lead`) to send the monthly digest to `user@example.com`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n workflow automation platform | [n8n Creator Partner Link](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Need an assessment on your business automation? | [Consultation Booking Page](https://khmuhtadin.com/consultation/) |