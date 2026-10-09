Manage farm box shares and weekly harvest emails with gpt-6-luna, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/manage-farm-box-shares-and-weekly-harvest-emails-with-gpt-6-luna--google-sheets-and-gmail-20443


# Manage farm box shares and weekly harvest emails with gpt-6-luna, Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates the administrative operations of a Community Supported Agriculture (CSA) farm box subscription service. It handles three core functional areas: processing member box change requests via an intake form, generating and broadcasting a weekly harvest newsletter every Thursday using AI, and conducting a weekly financial balance sweep every Monday with member reminders and a farm summary.

The workflow is categorized into the following logical blocks:
- **1.1 Input Reception & Validation:** Captures form submissions for box change requests, normalizes data fields, generates unique tracking identifiers, and validates mandatory inputs.
- **1.2 AI Processing & Request Logging:** Leverages an OpenAI language model to classify requests, decide on confirmation status, and draft reply emails, which are subsequently logged in Google Sheets and sent to members. Exception handling routes incomplete submissions directly to the farm inbox.
- **1.3 Weekly Harvest Newsletter:** Triggers weekly on Thursdays, aggregates active member counts and harvest inventory, employs an AI assistant to compose a storage-tip-enriched newsletter, dispatches the email to active members, and logs the distribution.
- **1.4 Weekly Balance Sweep:** Triggers weekly on Mondays, reviews member balances, issues reminders to accounts with positive balances, and summarizes outstanding balances for the farm administrators.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
This block initializes the processing of member change requests submitted through a hosted web form, normalizes disparate payload schemas, stamps metadata, and performs gatekeeping validation.

- **Nodes Involved:** `Box Change Request Form`, `Normalize Request`, `Validate Request`, `Email Farm About Invalid Request`

- **Node Details:**
  - **Box Change Request Form**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Webhook / Trigger)
    - *Configuration Choices:* Exposes a hosted web form titled "Farm box change request" requiring member name, email, change description, and an optional effective date.
    - *Key Expressions/Variables:* `webhookId: "farmbox-change"`
    - *Connections:* Input: None (Trigger); Output: `Normalize Request`
    - *Edge Cases / Potential Failures:* Form submission network failures or invalid webhook routing.
  - **Normalize Request**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation)
    - *Configuration Choices:* Maps incoming fields regardless of casing (e.g., camelCase vs. Form labels), populates `received_at` with an ISO timestamp, assigns a generated `request_id`, and sets a static farm contact email.
    - *Key Expressions/Variables:* `={{ $now.toISO() }}`, `={{ 'REQ-' + $now.toFormat('yyyyMMdd') + '-' + (100 + Math.floor(Math.random() * 900)) }}`
    - *Connections:* Input: `Box Change Request Form`; Output: `Validate Request`
    - *Edge Cases / Potential Failures:* Expression evaluation failures if payload properties are completely absent.
  - **Validate Request**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional routing)
    - *Configuration Choices:* Evaluates whether `member_name`, `member_email`, and `request` fields contain non-empty string values.
    - *Key Expressions/Variables:* `={{ $json.member_name }}`, `={{ $json.member_email }}`, `={{ $json.request }}`
    - *Connections:* Input: `Normalize Request`; Outputs: True branch to `Classify and Draft Reply`, False branch to `Email Farm About Invalid Request`
    - *Edge Cases / Potential Failures:* Loose type validation might pass unexpected data types.
  - **Email Farm About Invalid Request**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email integration)
    - *Configuration Choices:* Sends a plaintext notification email to the configured farm email detailing missing submission fields.
    - *Key Expressions/Variables:* `={{ $json.farm_email }}`, `={{ 'Farm box change request needs details from ' + ($json.member_name || 'a member') }}`
    - *Connections:* Input: `Validate Request` (False branch); Output: None (Terminal)
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`)
    - *Edge Cases / Potential Failures:* SMTP or OAuth authentication token expiration.

---

#### 2.2 AI Processing & Request Logging
Valid requests are evaluated by an OpenAI model to classify intent and draft a tailored response. The structured result is parsed, recorded in a Google Sheet, and sent to the member.

- **Nodes Involved:** `Classify and Draft Reply`, `Reply Model`, `Parse Reply`, `Log Request to Register`, `Email Confirmation to Member`

- **Node Details:**
  - **Classify and Draft Reply**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration Choices:* Configured with a system prompt instructing the model to classify requests (`skip_week`, `pause_membership`, `box_size_change`, `pickup_change`, `other`), determine a decision (`confirmed` or `needs_review`), and draft a plain-text confirmation email under 120 words.
    - *Key Expressions/Variables:* `={{ 'Member: ' + $json.member_name + ' | Request: ' + $json.request + ' | Effective date: ' + $json.effective_date }}`
    - *Connections:* Input: `Validate Request` (True branch), `Reply Model`; Output: `Parse Reply`
    - *Edge Cases / Potential Failures:* LLM token limits, API rate limits, or non-JSON output compliance failures.
  - **Reply Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model configuration)
    - *Configuration Choices:* Connects to OpenAI using the `gpt-6-luna` model identifier.
    - *Connections:* Input: None; Output: Linked to `Classify and Draft Reply` (`ai_languageModel`)
    - *Credentials:* OpenAI API (`jonathan`)
    - *Edge Cases / Potential Failures:* Invalid API keys or insufficient account credits.
  - **Parse Reply**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript data transformation)
    - *Configuration Choices:* Parses raw string text outputs from the AI agent into structured JSON properties (`request_type`, `decision`, `email_subject`, `email_body`), falling back safely if JSON parsing fails.
    - *Key Expressions/Variables:* Accesses `Normalize Request` parent payload via `$('Normalize Request').first()`.
    - *Connections:* Input: `Classify and Draft Reply`; Output: `Log Request to Register`
    - *Edge Cases / Potential Failures:* Malformed JSON generation by the LLM requiring fallback execution paths.
  - **Log Request to Register**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet logging)
    - *Configuration Choices:* Appends a new row to the `Requests` spreadsheet tab mapping request IDs, member details, classifications, decisions, and drafted email contents.
    - *Connections:* Input: `Parse Reply`; Output: `Email Confirmation to Member`
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`)
    - *Edge Cases / Potential Failures:* Sheet schema mismatches or API write quotas.
  - **Email Confirmation to Member**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email delivery)
    - *Configuration Choices:* Sends the finalized plain-text confirmation email directly to the requesting member.
    - *Key Expressions/Variables:* `={{ $('Parse Reply').first().json.member_email }}`, `={{ $('Parse Reply').first().json.email_body }}`
    - *Connections:* Input: `Log Request to Register`; Output: None (Terminal)
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`)
    - *Edge Cases / Potential Failures:* Invalid recipient email addresses or mail server bounces.

---

#### 2.3 Weekly Harvest Newsletter
Triggered every Thursday at 07:00, this block reads active member data and current harvest details, generates an AI-written newsletter with storage tips, and distributes it to eligible members.

- **Nodes Involved:** `Weekly Harvest Newsletter`, `Read Active Members`, `Collapse Members`, `Read This Week's Harvest`, `Collapse Harvest`, `Compose Harvest Newsletter`, `Newsletter Model`, `Prepare Newsletter Email`, `Email Members the Newsletter`, `Build Newsletter Log`, `Log Newsletter Sent`

- **Node Details:**
  - **Weekly Harvest Newsletter**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron Scheduler)
    - *Configuration Choices:* Configured to trigger weekly using a cron expression (`0 7 * * 4`, every Thursday at 07:00).
    - *Connections:* Input: None; Output: `Read Active Members`
  - **Read Active Members**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet read operation)
    - *Configuration Choices:* Reads all rows from the `Members` spreadsheet tab.
    - *Connections:* Input: `Weekly Harvest Newsletter`; Output: `Collapse Members`
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`)
  - **Collapse Members**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data aggregation)
    - *Configuration Choices:* Computes the total count of fetched roster rows.
    - *Connections:* Input: `Read Active Members`; Output: `Read This Week's Harvest`
  - **Read This Week's Harvest**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet read operation)
    - *Configuration Choices:* Reads inventory details from the `Harvest` spreadsheet tab.
    - *Connections:* Input: `Collapse Members`; Output: `Collapse Harvest`
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`)
  - **Collapse Harvest**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data transformation)
    - *Configuration Choices:* Filters active harvest items, formats them into a bulleted text list, and determines the target week string.
    - *Connections:* Input: `Read This Week's Harvest`; Output: `Compose Harvest Newsletter`
  - **Compose Harvest Newsletter**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration Choices:* System prompt directs the grower model to generate a warm newsletter under 180 words containing item storage tips based strictly on provided inventory items.
    - *Key Expressions/Variables:* `={{ 'Week of: ' + $json.week_of + ' | This week we harvested: ' + $json.harvest_text }}`
    - *Connections:* Input: `Collapse Harvest`, `Newsletter Model`; Output: `Prepare Newsletter Email`
    - *Edge Cases / Potential Failures:* Hallucinated inventory items if prompt constraints are breached.
  - **Newsletter Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model configuration)
    - *Configuration Choices:* Connects to OpenAI using the `gpt-6-luna` model identifier.
    - *Connections:* Input: None; Output: Linked to `Compose Harvest Newsletter` (`ai_languageModel`)
    - *Credentials:* OpenAI API (`jonathan`)
  - **Prepare Newsletter Email**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data fan-out transformer)
    - *Configuration Choices:* Filters out members whose `share_status` is marked as `paused`, parses the generated newsletter JSON, and fans out individual recipient message items for every active subscriber.
    - *Connections:* Input: `Compose Harvest Newsletter`; Output: `Email Members the Newsletter`
  - **Email Members the Newsletter**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email broadcasting)
    - *Configuration Choices:* Iterates through fanned-out items to dispatch the personalized newsletter email to each member.
    - *Key Expressions/Variables:* `={{ $json.member_email }}`, `={{ $json.email_body }}`
    - *Connections:* Input: `Prepare Newsletter Email`; Output: `Build Newsletter Log`
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`)
  - **Build Newsletter Log**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data transformation)
    - *Configuration Choices:* Generates audit record objects containing timestamps, weeks, recipient emails, and subjects for each successfully sent email.
    - *Connections:* Input: `Email Members the Newsletter`; Output: `Log Newsletter Sent`
  - **Log Newsletter Sent**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet logging)
    - *Configuration Choices:* Appends audit records to the `NewsletterLog` spreadsheet tab.
    - *Connections:* Input: `Build Newsletter Log`; Output: None (Terminal)
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`)

---

#### 2.4 Weekly Balance Sweep
Triggered every Monday at 08:00, this block reviews member account balances, filters accounts with outstanding amounts due, issues individual balance reminders, and sends an aggregated summary report to the farm administrators.

- **Nodes Involved:** `Weekly Balance Sweep`, `Read Member Balances`, `Find Outstanding Balances`, `Email Balance Reminders`, `Email Farm Balance Summary`

- **Node Details:**
  - **Weekly Balance Sweep**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron Scheduler)
    - *Configuration Choices:* Configured to trigger weekly using a cron expression (`0 8 * * 1`, every Monday at 08:00).
    - *Connections:* Input: None; Output: `Read Member Balances`
  - **Read Member Balances**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet read operation)
    - *Configuration Choices:* Reads roster and financial balance data from the `Members` spreadsheet tab.
    - *Connections:* Input: `Weekly Balance Sweep`; Output: `Find Outstanding Balances`
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`)
  - **Find Outstanding Balances**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data filtering and template builder)
    - *Configuration Choices:* Sanitizes balance values, filters for records where `balance_due` exceeds 0, and constructs individualized reminder bodies along with a consolidated farm summary payload.
    - *Connections:* Input: `Read Member Balances`; Output: `Email Balance Reminders`
  - **Email Balance Reminders**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email delivery)
    - *Configuration Choices:* Iterates through outstanding accounts to send payment reminder emails.
    - *Key Expressions/Variables:* `={{ $json.member_email }}`, `={{ $json.reminder_body }}`
    - *Connections:* Input: `Find Outstanding Balances`; Output: `Email Farm Balance Summary`
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`)
  - **Email Farm Balance Summary**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email notification)
    - *Configuration Choices:* Sends a consolidated summary email to the farm administrator listing all members with outstanding balances.
    - *Key Expressions/Variables:* `={{ $('Find Outstanding Balances').first().json.farm_email }}`, `={{ $('Find Outstanding Balances').first().json.summary_body }}`
    - *Connections:* Input: `Email Balance Reminders`; Output: None (Terminal)
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`)

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation overview & setup instructions | None | None | ## manage farm box shares and send the weekly harvest email with AI... |
| 1. Box change requests | n8n-nodes-base.stickyNote | Visual grouping block for request handling | None | None | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| 2. Weekly harvest newsletter | n8n-nodes-base.stickyNote | Visual grouping block for newsletter automation | None | None | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| 3. Weekly balance sweep | n8n-nodes-base.stickyNote | Visual grouping block for account balance auditing | None | None | ## 3. Weekly balance sweep<br><br>Every Monday each member balance is checked and friendly reminders go to members who owe, with a summary to the farm inbox. |
| Box Change Request Form | n8n-nodes-base.formTrigger | Captures member box change submissions | None | Normalize Request | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Normalize Request | n8n-nodes-base.set | Normalizes fields and stamps timestamps/IDs | Box Change Request Form | Validate Request | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Validate Request | n8n-nodes-base.if | Verifies presence of mandatory input fields | Normalize Request | Classify and Draft Reply, Email Farm About Invalid Request | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Classify and Draft Reply | @n8n/n8n-nodes-langchain.agent | AI Agent classifies request and drafts reply | Validate Request, Reply Model | Parse Reply | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Reply Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides OpenAI language model for request agent | None | Classify and Draft Reply | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Parse Reply | n8n-nodes-base.code | Parses AI output JSON and merges context | Classify and Draft Reply | Log Request to Register | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Log Request to Register | n8n-nodes-base.googleSheets | Appends processed request to spreadsheet | Parse Reply | Email Confirmation to Member | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Email Confirmation to Member | n8n-nodes-base.gmail | Sends confirmation email to member | Log Request to Register | None | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Email Farm About Invalid Request | n8n-nodes-base.gmail | Notifies farm inbox of incomplete submission | Validate Request | None | ## 1. Box change requests<br><br>Members submit a change on the form. It is validated, classified by AI, logged to Sheets, and confirmed by email. |
| Weekly Harvest Newsletter | n8n-nodes-base.scheduleTrigger | Triggers newsletter generation every Thursday | None | Read Active Members | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Read Active Members | n8n-nodes-base.googleSheets | Fetches member roster from spreadsheet | Weekly Harvest Newsletter | Collapse Members | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Collapse Members | n8n-nodes-base.code | Counts active members fetched from sheet | Read Active Members | Read This Week's Harvest | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Read This Week's Harvest | n8n-nodes-base.googleSheets | Fetches current harvest items from sheet | Collapse Members | Collapse Harvest | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Collapse Harvest | n8n-nodes-base.code | Formats harvest rows into item lists | Read This Week's Harvest | Compose Harvest Newsletter | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Compose Harvest Newsletter | @n8n/n8n-nodes-langchain.agent | AI Agent composes weekly harvest newsletter | Collapse Harvest, Newsletter Model | Prepare Newsletter Email | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Newsletter Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides OpenAI language model for newsletter agent | None | Compose Harvest Newsletter | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Prepare Newsletter Email | n8n-nodes-base.code | Filters active members and fans out newsletter payloads | Compose Harvest Newsletter | Email Members the Newsletter | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Email Members the Newsletter | n8n-nodes-base.gmail | Broadcasts newsletter email to active members | Prepare Newsletter Email | Build Newsletter Log | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Build Newsletter Log | n8n-nodes-base.code | Constructs audit log objects for sent newsletters | Email Members the Newsletter | Log Newsletter Sent | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Log Newsletter Sent | n8n-nodes-base.googleSheets | Appends newsletter audit records to spreadsheet | Build Newsletter Log | None | ## 2. Weekly harvest newsletter<br><br>Every Thursday the member roster and the harvest list are read, an AI assistant writes the newsletter, and it goes to all active members. |
| Weekly Balance Sweep | n8n-nodes-base.scheduleTrigger | Triggers balance audit every Monday | None | Read Member Balances | ## 3. Weekly balance sweep<br><br>Every Monday each member balance is checked and friendly reminders go to members who owe, with a summary to the farm inbox. |
| Read Member Balances | n8n-nodes-base.googleSheets | Reads member balances from spreadsheet | Weekly Balance Sweep | Find Outstanding Balances | ## 3. Weekly balance sweep<br><br>Every Monday each member balance is checked and friendly reminders go to members who owe, with a summary to the farm inbox. |
| Find Outstanding Balances | n8n-nodes-base.code | Filters balances > 0 and builds reminder templates | Read Member Balances | Email Balance Reminders | ## 3. Weekly balance sweep<br><br>Every Monday each member balance is checked and friendly reminders go to members who owe, with a summary to the farm inbox. |
| Email Balance Reminders | n8n-nodes-base.gmail | Sends individual balance reminders to members | Find Outstanding Balances | Email Farm Balance Summary | ## 3. Weekly balance sweep<br><br>Every Monday each member balance is checked and friendly reminders go to members who owe, with a summary to the farm inbox. |
| Email Farm Balance Summary | n8n-nodes-base.gmail | Sends summary of outstanding balances to farm | Email Balance Reminders | None | ## 3. Weekly balance sweep<br><br>Every Monday each member balance is checked and friendly reminders go to members who owe, with a summary to the farm inbox. |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the workflow manually in n8n:

1. **Create the Form Trigger**
   - Add a **Form Trigger** node named `Box Change Request Form`.
   - Set Webhook ID to `farmbox-change`.
   - Configure form fields: `member_name` (Full name, Required), `member_email` (user@example.com, Required), `request` (Textarea, Required), `effective_date` (YYYY-MM-DD, Optional). Title: "Farm box change request".

2. **Normalize Incoming Data**
   - Add a **Set** node named `Normalize Request`.
   - Connect `Box Change Request Form` output to `Normalize Request`.
   - Create string assignments for `member_name`, `member_email`, `request`, and `effective_date` using fallback expressions (e.g., `={{ $json.member_name || $json['Member name'] || '' }}`). Add `received_at` (`={{ $now.toISO() }}`), `request_id` (`={{ 'REQ-' + $now.toFormat('yyyyMMdd') + '-' + (100 + Math.floor(Math.random() * 900)) }}`), and `farm_email` (`user@example.com`).

3. **Validate Form Submission**
   - Add an **If** node named `Validate Request`.
   - Connect `Normalize Request` to `Validate Request`.
   - Configure conditions (AND combinator) to check that `member_name`, `member_email`, and `request` are not empty.

4. **Handle Invalid Submissions**
   - Add a **Gmail** node named `Email Farm About Invalid Request`.
   - Connect the `false` output of `Validate Request` to this node.
   - Configure credentials (`Gmail Fresh Sep05`), recipient `={{ $json.farm_email }}`, and set text body detailing missing fields.

5. **Configure AI Request Classification Agent**
   - Add an **AI Agent** node named `Classify and Draft Reply`. Connect the `true` output of `Validate Request` to its main input.
   - Set prompt type to define, text expression to `={{ 'Member: ' + $json.member_name + ' | Request: ' + $json.request + ' | Effective date: ' + $json.effective_date }}`, and apply the system prompt instructing JSON output with keys: `request_type`, `decision`, `subject`, `body`.
   - Add an **OpenAI Chat Model** node named `Reply Model` (configured with model `gpt-6-luna` and OpenAI credentials) and connect its output to the agent's `ai_languageModel` input.

6. **Parse AI Response**
   - Add a **Code** node named `Parse Reply`. Connect `Classify and Draft Reply` output here.
   - Insert JavaScript to safely parse the AI string output into structured JSON combining base request data and parsed properties (`request_type`, `decision`, `email_subject`, `email_body`).

7. **Log and Confirm Request**
   - Add a **Google Sheets** node named `Log Request to Register`. Connect `Parse Reply` output here.
   - Configure credentials (`Feedback Analyzer Sheets`), spreadsheet ID/Name, operation `append`, sheet `Requests`, and map spreadsheet columns (`request_id`, `received_at`, `member_name`, `member_email`, `request_type`, `effective_date`, `details`, `decision`, `reply_subject`, `reply_body`) to corresponding `$json` properties.
   - Add a **Gmail** node named `Email Confirmation to Member`. Connect `Log Request to Register` to this node. Configure recipient `={{ $('Parse Reply').first().json.member_email }}`, subject `={{ $('Parse Reply').first().json.email_subject }}`, and message `={{ $('Parse Reply').first().json.email_body }}`.

8. **Build Weekly Harvest Newsletter Pipeline**
   - Add a **Schedule Trigger** node named `Weekly Harvest Newsletter` set to cron expression `0 7 * * 4`.
   - Add a **Google Sheets** node named `Read Active Members` (sheet `Members`) connected to the schedule trigger.
   - Add a **Code** node named `Collapse Members` to calculate active member counts.
   - Add a **Google Sheets** node named `Read This Week's Harvest` (sheet `Harvest`) connected from `Collapse Members`.
   - Add a **Code** node named `Collapse Harvest` to format harvest rows.
   - Add an **AI Agent** node named `Compose Harvest Newsletter` connected from `Collapse Harvest`, paired with an **OpenAI Chat Model** node named `Newsletter Model` (`gpt-6-luna`).
   - Add a **Code** node named `Prepare Newsletter Email` to filter out paused members and fan out individual recipient payloads.
   - Add a **Gmail** node named `Email Members the Newsletter` to send the emails.
   - Add a **Code** node named `Build Newsletter Log` to generate audit entries.
   - Add a **Google Sheets** node named `Log Newsletter Sent` (sheet `NewsletterLog`) to append the audit records.

9. **Build Weekly Balance Sweep Pipeline**
   - Add a **Schedule Trigger** node named `Weekly Balance Sweep` set to cron expression `0 8 * * 1`.
   - Add a **Google Sheets** node named `Read Member Balances` (sheet `Members`) connected to the schedule trigger.
   - Add a **Code** node named `Find Outstanding Balances` to filter records where `balance_due > 0` and construct reminder and summary messages.
   - Add a **Gmail** node named `Email Balance Reminders` to send reminders to members.
   - Add a **Gmail** node named `Email Farm Balance Summary` connected from `Email Balance Reminders` to send the consolidated balance summary to the farm inbox (`user@example.com`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n workflow automation platform | [n8n Platform](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Business consultation and project assessment services | [Consultation Link](https://khmuhtadin.com/consultation/) |