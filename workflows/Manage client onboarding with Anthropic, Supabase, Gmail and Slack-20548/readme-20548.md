Manage client onboarding with Anthropic, Supabase, Gmail and Slack

https://n8nworkflows.xyz/workflows/manage-client-onboarding-with-anthropic--supabase--gmail-and-slack-20548


# Manage client onboarding with Anthropic, Supabase, Gmail and Slack

### 1. Workflow Overview

The **Client Onboarding Automation System** workflow is designed for agencies and service-oriented businesses to fully automate their client onboarding lifecycle. The primary purpose is to eliminate manual overhead associated with intake processing, file organization, initial email communications, CRM/database entry, team notifications, and ongoing task verification.

The workflow supports target use cases including automated client data normalization, AI-driven project scope and priority evaluation, cloud folder provisioning, personalized welcome messaging, internal team alerting, and automated follow-up reminders based on task completion tracking.

The system logic is organized into four consecutive functional blocks:
- **1.1 Intake & Analyze:** Receives client submissions, sanitizes input data, leverages an Anthropic language model to evaluate requirements, prioritize the project, and check for missing information.
- **1.2 Create Records & Welcome:** Evaluates validation checks to either request missing data or provision database records in Supabase, generate a Google Drive folder, draft and send an AI-personalized welcome email, notify the team via Slack, and update client statuses.
- **1.3 Wait & Check Progress:** Pauses the execution thread for a configured duration before querying the backend database to audit the status of the client's onboarding task checklist.
- **1.4 Complete or Remind:** Evaluates whether all onboarding milestones have been achieved, transitioning the client status to completed if successful or dispatching a targeted reminder listing pending action items if incomplete.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Analyze
- **Overview:** This block acts as the entry point for raw client onboarding submissions. It cleans and standardizes the incoming form data, uses an advanced AI agent to analyze requirements, extract metadata, estimate priorities, and detect missing information to determine downstream routing.
- **Nodes Involved:**
  - `Client Onboarding Form`
  - `Clean Client Data`
  - `AI - Analyze Client Requirements`
  - `Anthropic Chat Model`
  - `Check Required Information`

- **Node Details:**
  - **Client Onboarding Form**
    - *Type and technical role:* `n8n-nodes-base.formTrigger` (Trigger Node). Listens for incoming POST requests submitted via an n8n-hosted HTML form.
    - *Configuration choices:* Configured with specific form fields including Company Name, Contact Name, Email, Phone, Website, Service Required (Dropdown), CRM, Requirements (Textarea), and Expected Launch Date.
    - *Key expressions or variables:* None (generates the initial trigger payload).
    - *Input and output connections:* Inputs: None (Entrypoint). Output: Connects to `Clean Client Data`.
    - *Version-specific requirements:* Type version 2.2.
    - *Edge cases or potential failure types:* Form submission interruptions or invalid webhooks.

  - **Clean Client Data**
    - *Type and technical role:* `n8n-nodes-base.set` (Data Transformation Node). Normalizes, trims, and formats raw input fields while defining global configuration variables like the agency name.
    - *Configuration choices:* Sets multiple assignment variables mapping raw form keys to clean snake_case variables (e.g., `agency_name`, `company_name`, `contact_name`, `email`, `phone`, `website`, `service_required`, `crm`, `requirements`, `expected_launch_date`, `submitted_at`).
    - *Key expressions or variables:* Uses JavaScript string manipulation methods (e.g., `String(...).trim()`, `.toLowerCase()`, regex substitution for phone formatting, and URL protocol checks for the website). Includes `$now.toISO()` for submission timestamps.
    - *Input and output connections:* Input: `Client Onboarding Form`. Output: Connects to `AI - Analyze Client Requirements`.
    - *Version-specific requirements:* Type version 3.4.
    - *Edge cases or potential failure types:* Missing optional fields can yield empty strings; handled via fallback logical operators.

  - **AI - Analyze Client Requirements**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent Node). Interprets unstructured client requirements using Anthropic to extract structured analytical insights.
    - *Configuration choices:* Configured with a system message enforcing strict JSON output containing service normalization, priority calculation, CRM extraction, restated requirements, missing information array, and an internal onboarding summary. Includes error handling (`onError: "continueRegularOutput"`) and retries up to 3 times.
    - *Key expressions or variables:* Passes a JSON string containing cleaned data from `Clean Client Data` and the ISO date from `$now.toISODate()`.
    - *Input and output connections:* Input: `Clean Client Data`, `Anthropic Chat Model` (AI Model Link). Output: Connects to `Check Required Information`.
    - *Version-specific requirements:* Type version 1.7.
    - *Edge cases or potential failure types:* AI output parsing issues or API timeout errors; mitigated by retry limits and error fallbacks.

  - **Anthropic Chat Model**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.lmChatAnthropic` (Language Model Provider Node). Provides the LLM backend for the AI agent.
    - *Configuration choices:* Uses model `claude-haiku-4-5-20251001` with a temperature setting of `0.3`.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Output links to `AI - Analyze Client Requirements` via the `ai_languageModel` connection.
    - *Version-specific requirements:* Type version 1.3.
    - *Edge cases or potential failure types:* Authentication credential failure or API rate limits.

  - **Check Required Information**
    - *Type and technical role:* `n8n-nodes-base.if` (Conditional Branching Node). Validates whether essential contact and requirement parameters are present.
    - *Configuration choices:* Evaluates conditions combining logical `AND` rules to check that `company_name`, `contact_name`, `email`, `phone`, and `requirements` are not empty.
    - *Key expressions or variables:* References data from `Clean Client Data`.
    - *Input and output connections:* Input: `AI - Analyze Client Requirements`. Outputs: True branch connects to `Create Client Record`; False branch connects to `Send Onboarding Reminder`.
    - *Version-specific requirements:* Type version 2.
    - *Edge cases or potential failure types:* Empty evaluation if upstream data is unpopulated.

---

#### 2.2 Create Records & Welcome
- **Overview:** Executes when all required information is confirmed. It registers the client and checklist in Supabase, provisions a dedicated Google Drive folder, drafts a personalized welcome email via AI, dispatches the email via Gmail, notifies the internal team on Slack, and updates the client status to active onboarding.
- **Nodes Involved:**
  - `Create Client Record`
  - `Create Onboarding Checklist`
  - `Create Client Folder`
  - `Generate Welcome Message`
  - `Anthropic Chat Model1`
  - `Send Welcome Email`
  - `Notify Internal Team`
  - `Update Client Status`

- **Node Details:**
  - **Create Client Record**
    - *Type and technical role:* `n8n-nodes-base.supabase` (Database Operation Node). Inserts a new client row into the Supabase `clients` table.
    - *Configuration choices:* Maps incoming structured attributes from `Clean Client Data` and parsed outputs from `AI - Analyze Client Requirements` (falling back safely if JSON parsing encounters malformed data). Sets initial status to `onboarding`.
    - *Key expressions or variables:* Uses expressions to safely parse JSON strings from `AI - Analyze Client Requirements` using `JSON.parse(...)` inside try-catch blocks.
    - *Input and output connections:* Input: `Check Required Information` (True branch). Output: Connects to `Create Onboarding Checklist`.
    - *Version-specific requirements:* Type version 1.
    - *Edge cases or potential failure types:* Database schema mismatches, missing required table columns, or constraint violations.

  - **Create Onboarding Checklist**
    - *Type and technical role:* `n8n-nodes-base.supabase` (Database Operation Node). Inserts a tracking checklist record linked to the newly created client.
    - *Configuration choices:* Links `client_id` from the preceding database row insertion and sets default boolean flags (`false`) across all six onboarding tasks.
    - *Key expressions or variables:* `={{ $('Create Client Record').first().json.id }}`
    - *Input and output connections:* Input: `Create Client Record`. Output: Connects to `Create Client Folder`.
    - *Version-specific requirements:* Type version 1.
    - *Edge cases or potential failure types:* Foreign key constraint errors if the client record fails to return an ID.

  - **Create Client Folder**
    - *Type and technical role:* `n8n-nodes-base.googleDrive` (Cloud Storage Node). Creates a dedicated folder for the client inside Google Drive.
    - *Configuration choices:* Sets resource to `folder`, names the folder after the cleaned company name, and places it inside the specified parent folder ID. Configured with error continuity (`onError: "continueRegularOutput"`) and 3 retries.
    - *Key expressionsOrVariables:* `={{ $('Clean Client Data').first().json.company_name }}`
    - *Input and output connections:* Input: `Create Onboarding Checklist`. Output: Connects to `Generate Welcome Message`.
    - *Version-specific requirements:* Type version 3.
    - *Edge cases or potential failure types:* Insufficient OAuth scopes or invalid parent folder IDs; handled gracefully via error continuation settings.

  - **Generate Welcome Message**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent Node). Generates personalized welcome prose using Anthropic based on client context.
    - *Configuration choices:* System prompt enforces an 80-120 word professional welcome note without pricing or dates, outputting strictly raw JSON with `subject` and `message` keys. Includes retry logic and error continuation.
    - *Key expressions or variables:* Serializes agency data, contact names, services, requirements, and AI summaries into a JSON payload string.
    - *Input and output connections:* Input: `Create Client Folder`, `Anthropic Chat Model1` (AI Model Link). Output: Connects to `Send Welcome Email`.
    - *Version-specific requirements:* Type version 1.7.
    - *Edge cases or potential failure types:* Parsing failures from non-JSON AI output models; mitigated by fallback safety patterns.

  - **Anthropic Chat Model1**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.lmChatAnthropic` (Language Model Provider Node). Secondary language model instance supporting the welcome message agent.
    - *Configuration choices:* Uses model `claude-haiku-4-5-20251001` with a temperature of `0.3`.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Output links to `Generate Welcome Message` via the `ai_languageModel` connection.
    - *Version-specific requirements:* Type version 1.3.
    - *Edge cases or potential failure types:* API token quotas or connectivity timeouts.

  - **Send Welcome Email**
    - *Type and technical role:* `n8n-nodes-base.gmail` (Email Integration Node). Sends the fully compiled HTML welcome email to the client.
    - *Configuration choices:* Dynamically builds an HTML body incorporating the AI-generated message, client metadata, task lists, and missing information requirements. Sets custom subject lines dynamically.
    - *Key expressions or variables:* Complex JavaScript template assembling variables from `Clean Client Data`, `AI - Analyze Client Requirements`, and `Generate Welcome Message` with HTML escaping functions.
    - *Input and output connections:* Input: `Generate Welcome Message`. Output: Connects to `Notify Internal Team`.
    - *Version-specific requirements:* Type version 2.1.
    - *Edge cases or potential failure types:* Invalid email formats, Gmail OAuth token expiration, or daily sending quotas.

  - **Notify Internal Team**
    - *Type and technical role:* `n8n-nodes-base.slack` (Chat Notification Node). Posts a rich text summary of the new onboarding start to a Slack channel.
    - *Configuration choices:* Selects channel by name (`client-onboarding`) and sends a formatted markdown message summarizing the client profile, priority, Google Drive link, and AI evaluation summary.
    - *Key expressions or variables:* References data from `Clean Client Data`, `AI - Analyze Client Requirements`, and `Create Client Folder`.
    - *Input and output connections:* Input: `Send Welcome Email`. Output: Connects to `Update Client Status`.
    - *Version-specific requirements:* Type version 2.3.
    - *Edge cases or potential failure types:* Missing channel permissions or bot not invited to the target channel.

  - **Update Client Status**
    - *Type and technical role:* `n8n-nodes-base.supabase` (Database Operation Node). Updates the client row status in Supabase to reflect active onboarding.
    - *Configuration choices:* Filters by client record ID, updating `status` to `onboarding_started`, logging `onboarding_started_at`, and saving the `drive_folder_url`.
    - *Key expressions or variables:* Uses expressions to evaluate folder URL validity and current timestamps.
    - *Input and output connections:* Input: `Notify Internal Team`. Output: Connects to `Wait for Client Completion`.
    - *Version-specific requirements:* Type version 1.
    - *Edge cases or potential failure types:* Database connection issues or primary key lookup mismatches.

---

#### 2.3 Wait & Check Progress
- **Overview:** Pauses workflow execution for a configured duration to allow internal teams time to fulfill operational setup tasks, then queries the Supabase database to retrieve the latest checklist completion state.
- **Nodes Involved:**
  - `Wait for Client Completion`
  - `Check Onboarding Status`

- **Node Details:**
  - **Wait for Client Completion**
    - *Type and technical role:* `n8n-nodes-base.wait` (Flow Control Node). Suspends the workflow execution for a defined duration.
    - *Configuration choices:* Pauses execution for `1` day (configurable). Requires execution state persistence.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: `Update Client Status`. Output: Connects to `Check Onboarding Status`.
    - *Version-specific requirements:* Type version 1.1.
    - *Edge cases or potential failure types:* Execution pruning or server restarts before the wait resumes (requires proper workflow execution saving configuration).

  - **Check Onboarding Status**
    - *Type and technical role:* `n8n-nodes-base.supabase` (Database Operation Node). Retrieves the active onboarding checklist row associated with the client.
    - *Configuration choices:* Executes a `getAll` operation querying the `onboarding_checklists` table with a limit of 1, filtered by `client_id`. Configured with `alwaysOutputData: true`.
    - *Key expressions or variables:* `={{ $('Create Client Record').first().json.id }}`
    - *Input and output connections:* Input: `Wait for Client Completion`. Output: Connects to `Is Onboarding Complete?`.
    - *Version-specific requirements:* Type version 1.
    - *Edge cases or potential failure types:* Missing records or database connectivity loss.

---

#### 2.4 Complete or Remind
- **Overview:** Evaluates the checklist status returned from the database. If all tasks are completed, marks the client profile as completed and notifies the team. If tasks remain incomplete, dispatches a reminder email detailing the pending items.
- **Nodes Involved:**
  - `Is Onboarding Complete?`
  - `Complete Onboarding`
  - `Notify Team - Onboarding Complete`
  - `Send Onboarding Reminder`

- **Node Details:**
  - **Is Onboarding Complete?**
    - *Type and technical role:* `n8n-nodes-base.if` (Conditional Branching Node). Validates whether all checklist verification flags are set to true.
    - *Configuration choices:* Evaluates an array of all six checklist task keys using `.every(k => $json[k] === true)`.
    - *Key expressions or variables:* JavaScript Array `.every()` execution against node data payload keys.
    - *Input and output connections:* Input: `Check Onboarding Status`. Outputs: True branch connects to `Complete Onboarding`; False branch connects to `Send Onboarding Reminder`.
    - *Version-specific requirements:* Type version 2.
    - *Edge cases or potential failure types:* Malformed boolean property values in the database record.

  - **Complete Onboarding**
    - *Type and technical role:* `n8n-nodes-base.supabase` (Database Operation Node). Updates the client status in the database to completed.
    - *Configuration choices:* Filters by client ID, updating `status` to `completed` and setting `completed_at` to the current timestamp.
    - *Key expressions or variables:* Uses `$now.toISO()`.
    - *Input and output connections:* Input: `Is Onboarding Complete?` (True branch). Output: Connects to `Notify Team - Onboarding Complete`.
    - *Version-specific requirements:* Type version 1.
    - *Edge cases or potential failure types:* Database write errors.

  - **Notify Team - Onboarding Complete**
    - *Type and technical role:* `n8n-nodes-base.slack` (Chat Notification Node). Posts a completion notification to the team Slack channel.
    - *Configuration choices:* Sends a text payload to the `client-onboarding` channel confirming that all tasks are finished and the build can begin.
    - *Key expressions or variables:* References client details from `Clean Client Data` and parsed service definitions.
    - *Input and output connections:* Input: `Complete Onboarding`. Output: None (Terminal node).
    - *Version-specific requirements:* Type version 2.3.
    - *Edge cases or potential failure types:* Slack channel permissions issues.

  - **Send Onboarding Reminder**
    - *Type and technical role:* `n8n-nodes-base.gmail` (Email Integration Node). Sends an email requesting missing intake details or reminding the client of pending tasks.
    - *Configuration choices:* Dual-purpose node: handles initial missing-information requests (from `Check Required Information`) and post-wait pending-task follow-ups (from `Is Onboarding Complete?`). Dynamically adapts its content and subject line based on whether `Check Onboarding Status` was executed.
    - *Key expressions or variables:* Conditional JavaScript evaluation inspecting `.isExecuted` states to filter pending tasks versus missing form parameters.
    - *Input and output connections:* Inputs: `Check Required Information` (False branch) and `Is Onboarding Complete?` (False branch). Output: None (Terminal node).
    - *Version-specific requirements:* Type version 2.1.
    - *Edge cases or potential failure types:* Invalid recipient email configurations or SMTP rate limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Client Onboarding Form | n8n-nodes-base.formTrigger | Triggers workflow on form submission | None | Clean Client Data | **1. Intake & Analyze**<br>Collects the client's details via form, cleans the data, and uses AI to extract service type, priority, CRM, and anything still missing. Incomplete forms are routed straight to a reminder email instead of continuing. |
| Clean Client Data | n8n-nodes-base.set | Normalizes and sanitizes form inputs | Client Onboarding Form | AI - Analyze Client Requirements | **1. Intake & Analyze**<br>Collects the client's details via form, cleans the data, and uses AI to extract service type, priority, CRM, and anything still missing. Incomplete forms are routed straight to a reminder email instead of continuing. |
| AI - Analyze Client Requirements | @n8n/n8n-nodes-langchain.agent | Analyzes requirements with Anthropic | Clean Client Data | Check Required Information | **1. Intake & Analyze**<br>Collects the client's details via form, cleans the data, and uses AI to extract service type, priority, CRM, and anything still missing. Incomplete forms are routed straight to a reminder email instead of continuing. |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM provider for analysis agent | None | AI - Analyze Client Requirements | **1. Intake & Analyze**<br>Collects the client's details via form, cleans the data, and uses AI to extract service type, priority, CRM, and anything still missing. Incomplete forms are routed straight to a reminder email instead of continuing. |
| Check Required Information | n8n-nodes-base.if | Validates presence of mandatory info | AI - Analyze Client Requirements | Create Client Record, Send Onboarding Reminder | **1. Intake & Analyze**<br>Collects the client's details via form, cleans the data, and uses AI to extract service type, priority, CRM, and anything still missing. Incomplete forms are routed straight to a reminder email instead of continuing. |
| Create Client Record | n8n-nodes-base.supabase | Inserts client row into Supabase | Check Required Information | Create Onboarding Checklist | **2. Create Records & Welcome**<br>Creates the client and checklist records in Supabase, makes their Drive folder, AI drafts a personalized welcome email, sends it, notifies the team on Slack, and marks the client as onboarding-started. |
| Create Onboarding Checklist | n8n-nodes-base.supabase | Creates checklist row in Supabase | Create Client Record | Create Client Folder | **2. Create Records & Welcome**<br>Creates the client and checklist records in Supabase, makes their Drive folder, AI drafts a personalized welcome email, sends it, notifies the team on Slack, and marks the client as onboarding-started. |
| Create Client Folder | n8n-nodes-base.googleDrive | Creates client folder in Google Drive | Create Onboarding Checklist | Generate Welcome Message | **2. Create Records & Welcome**<br>Creates the client and checklist records in Supabase, makes their Drive folder, AI drafts a personalized welcome email, sends it, notifies the team on Slack, and marks the client as onboarding-started. |
| Generate Welcome Message | @n8n/n8n-nodes-langchain.agent | Generates custom welcome email text | Create Client Folder | Send Welcome Email | **2. Create Records & Welcome**<br>Creates the client and checklist records in Supabase, makes their Drive folder, AI drafts a personalized welcome email, sends it, notifies the team on Slack, and marks the client as onboarding-started. |
| Anthropic Chat Model1 | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM provider for welcome message agent | None | Generate Welcome Message | **2. Create Records & Welcome**<br>Creates the client and checklist records in Supabase, makes their Drive folder, AI drafts a personalized welcome email, sends it, notifies the team on Slack, and marks the client as onboarding-started. |
| Send Welcome Email | n8n-nodes-base.gmail | Sends welcome email to client | Generate Welcome Message | Notify Internal Team | **2. Create Records & Welcome**<br>Creates the client and checklist records in Supabase, makes their Drive folder, AI drafts a personalized welcome email, sends it, notifies the team on Slack, and marks the client as onboarding-started. |
| Notify Internal Team | n8n-nodes-base.slack | Posts onboarding start alert to Slack | Send Welcome Email | Update Client Status | **2. Create Records & Welcome**<br>Creates the client and checklist records in Supabase, makes their Drive folder, AI drafts a personalized welcome email, sends it, notifies the team on Slack, and marks the client as onboarding-started. |
| Update Client Status | n8n-nodes-base.supabase | Updates client status to started | Notify Internal Team | Wait for Client Completion | **2. Create Records & Welcome**<br>Creates the client and checklist records in Supabase, makes their Drive folder, AI drafts a personalized welcome email, sends it, notifies the team on Slack, and marks the client as onboarding-started. |
| Wait for Client Completion | n8n-nodes-base.wait | Pauses workflow for reminder delay | Update Client Status | Check Onboarding Status | **3. Wait & Check Progress**<br>Pauses (default 1 days), then checks the Supabase checklist to see if all 6 onboarding tasks are marked done. |
| Check Onboarding Status | n8n-nodes-base.supabase | Queries checklist status from database | Wait for Client Completion | Is Onboarding Complete? | **3. Wait & Check Progress**<br>Pauses (default 1 days), then checks the Supabase checklist to see if all 6 onboarding tasks are marked done. |
| Is Onboarding Complete? | n8n-nodes-base.if | Evaluates if all checklist tasks are true | Check Onboarding Status | Complete Onboarding, Send Onboarding Reminder | **4. Complete or Remind**<br>If every task is done, marks the client complete and notifies the team. If not, sends the client a reminder listing exactly what's still pending. |
| Complete Onboarding | n8n-nodes-base.supabase | Marks client status as completed | Is Onboarding Complete? | Notify Team - Onboarding Complete | **4. Complete or Remind**<br>If every task is done, marks the client complete and notifies the team. If not, sends the client a reminder listing exactly what's still pending. |
| Notify Team - Onboarding Complete | n8n-nodes-base.slack | Notifies team of completion on Slack | Complete Onboarding | None | **4. Complete or Remind**<br>If every task is done, marks the client complete and notifies the team. If not, sends the client a reminder listing exactly what's still pending. |
| Send Onboarding Reminder | n8n-nodes-base.gmail | Emails missing data request or task reminder | Check Required Information, Is Onboarding Complete? | None | **4. Complete or Remind**<br>If every task is done, marks the client complete and notifies the team. If not, sends the client a reminder listing exactly what's still pending. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually:

1. **Create the Entry Trigger Node:**
   - Add a **Form Trigger** (`n8n-nodes-base.formTrigger`) named `Client Onboarding Form`.
   - Configure form fields: *Company Name* (Required), *Contact Name* (Required), *Email* (Email, Required), *Phone*, *Website*, *Service Required* (Dropdown with options: `WhatsApp Automation`, `AI Voice Agent`, `CRM Automation`, `Lead Generation & Outreach`, `n8n Workflow Development`, `Other`, Required), *CRM*, *Requirements* (Textarea), and *Expected Launch Date* (Date).

2. **Add Data Cleansing Node:**
   - Add a **Set** (`n8n-nodes-base.set`) node named `Clean Client Data`. Connect it to the Form Trigger.
   - Configure assignments to sanitize inputs (trim strings, normalize emails to lowercase, strip invalid phone characters, ensure website URLs include `https://`, and set `agency_name` to your preferred agency name).

3. **Configure AI Requirement Analysis:**
   - Add an **Advanced AI Agent** (`@n8n/n8n-nodes-langchain.agent`) node named `AI - Analyze Client Requirements`. Connect it to `Clean Client Data`.
   - Set prompt type to define, provide a system message instructing the model to output a strict raw JSON object with keys: `service`, `priority`, `crm`, `requirements`, `missing_information`, and `onboarding_summary`.
   - Add an **Anthropic Chat Model** (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) node, select model `claude-haiku-4-5-20251001` with temperature `0.3`, and connect it via the AI model port. Configure required Anthropic credentials.

4. **Add Input Validation Branch:**
   - Add an **If** (`n8n-nodes-base.if`) node named `Check Required Information`. Connect it to `AI - Analyze Client Requirements`.
   - Set conditions to verify that `company_name`, `contact_name`, `email`, `phone`, and `requirements` are not empty.

5. **Build the Success Branch (Database & Storage):**
   - **Create Client Record:** Add a **Supabase** (`n8n-nodes-base.supabase`) node named `Create Client Record`. Connect the True output of `Check Required Information` to this node. Configure table `clients` and map fields using expressions parsing the AI JSON output. Connect Supabase credentials.
   - **Create Checklist:** Add a second **Supabase** node named `Create Onboarding Checklist` connected after the client record node. Target table `onboarding_checklists`, linking `client_id` and setting all 6 task checklist columns to `false`.
   - **Create Drive Folder:** Add a **Google Drive** (`n8n-nodes-base.googleDrive`) node named `Create Client Folder` connected after the checklist node. Set resource to `folder`, name to `={{ $('Clean Client Data').first().json.company_name }}`, and select your parent "Clients" folder. Connect Google Drive OAuth2 credentials.

6. **Configure Welcome Message Generation & Dispatch:**
   - **Generate Welcome Message:** Add an **Advanced AI Agent** node named `Generate Welcome Message` connected after `Create Client Folder`. Set system message to generate an 80-120 word professional welcome email returning a JSON object with `subject` and `message`.
   - **Anthropic Model:** Add a second **Anthropic Chat Model** node configured similarly to the first and connect it to the welcome message agent.
   - **Send Welcome Email:** Add a **Gmail** (`n8n-nodes-base.gmail`) node named `Send Welcome Email` connected after the welcome message agent. Use expressions to construct the HTML email body and subject line. Connect Gmail OAuth2 credentials.

7. **Team Notification & Status Update:**
   - **Slack Notification:** Add a **Slack** (`n8n-nodes-base.slack`) node named `Notify Internal Team` connected after `Send Welcome Email`. Configure OAuth2 credentials, select channel `client-onboarding`, and populate the markdown message body.
   - **Update Status:** Add a Supabase node named `Update Client Status` connected after Slack. Filter by record ID and update status to `onboarding_started`.

8. **Wait & Follow-Up Verification Branch:**
   - **Wait Node:** Add a **Wait** (`n8n-nodes-base.wait`) node named `Wait for Client Completion` connected after the status update node. Configure duration for 1 day.
   - **Check Status:** Add a Supabase node named `Check Onboarding Status` connected after the wait node. Query `onboarding_checklists` filtering by `client_id`.
   - **Completion Condition:** Add an **If** node named `Is Onboarding Complete?` connected after status check. Set condition to check that all six checklist tasks evaluate to `true`.
   - **Final Completion:** Connect the True branch to a Supabase node named `Complete Onboarding` (updates status to `completed`), followed by a Slack node named `Notify Team - Onboarding Complete`.
   - **Reminder Branch:** Connect the False branch of `Check Required Information` and the False branch of `Is Onboarding Complete?` to a shared **Gmail** node named `Send Onboarding Reminder` configured to send missing info requests or pending task reminders.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Client Onboarding Automation System Purpose | Designed for automated client intake, record provisioning, folder creation, email dispatch, team alerts, and checklist tracking. |
| Supabase Table Requirements | Requires `clients` and `onboarding_checklists` tables configured with matching schema fields. |
| Slack Channel Requirement | Ensure the target Slack channel (`client-onboarding`) exists and the Slack bot integration has been invited to it. |