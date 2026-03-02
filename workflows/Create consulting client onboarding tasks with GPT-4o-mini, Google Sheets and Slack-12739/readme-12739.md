Create consulting client onboarding tasks with GPT-4o-mini, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/create-consulting-client-onboarding-tasks-with-gpt-4o-mini--google-sheets-and-slack-12739


# Create consulting client onboarding tasks with GPT-4o-mini, Google Sheets and Slack

## 1. Workflow Overview

**Workflow name:** Automated Client Onboarding and Task Assignment System  
**Provided title:** Create consulting client onboarding tasks with GPT-4o-mini, Google Sheets and Slack

**Purpose & use cases**  
This workflow receives new client details (via a webhook intake form), normalizes the data, uses **GPT‑4o‑mini** to generate a tailored onboarding checklist/tasks/deliverables, routes the client by project type (**Strategy / Management / IT**), logs the client to the appropriate **Google Sheets** tab, notifies the correct team via **Slack**, sends the client a **welcome email**, schedules a **Google Calendar kickoff meeting**, and finally syncs the record to a **CRM API**.

### 1.1 Input Reception & Global Config
Receives the form submission and injects environment/workflow constants (Slack channel IDs, Google Sheet ID, CRM URL).

### 1.2 Data Normalization
Maps multiple possible input field names into a consistent internal schema (clientName, clientEmail, projectType, etc.).

### 1.3 AI Generation
Calls OpenAI (GPT‑4o‑mini) to produce a JSON payload: checklist, firstTasks, deliverables.

### 1.4 Routing + Logging + Internal Notifications
Switches by projectType → appends/updates the correct Google Sheet tab → sends Slack message to the assigned consultant channel.

### 1.5 Client Communication + Scheduling + CRM Sync
Sends client welcome email (including checklist/tasks), creates a kickoff meeting event, then posts the consolidated payload to the CRM endpoint.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Global Config
**Overview:** Accepts inbound client submissions and defines workflow-level configuration values used later (Slack channel IDs, sheet ID, CRM URL).  
**Nodes involved:** `New Client Intake Form`, `Workflow Configuration`

#### Node: New Client Intake Form
- **Type / role:** Webhook (entry point) — receives the intake payload.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `client-intake`
  - **Response mode:** `lastNode` (the HTTP response to the caller will be whatever the final executed node returns).
- **Input/Output:**
  - **Input:** External HTTP POST body (expected fields like `name`, `email`, `projectType`, etc.).
  - **Output:** Emits an item with at least `$json.body` containing submitted fields.
  - **Next node:** `Workflow Configuration`
- **Edge cases / failures:**
  - Missing/invalid body fields → later expressions may produce empty strings.
  - If the workflow errors, the webhook caller may receive an error (since response is last node).
  - If multiple clients are posted in one request (array payload), normalization may not behave as intended unless the form always sends a single object.

#### Node: Workflow Configuration
- **Type / role:** Set node — injects configuration constants.
- **Key configuration choices:**
  - Adds these fields (placeholders must be replaced):
    - `strategyConsultantChannel`
    - `managementConsultantChannel`
    - `itConsultantChannel`
    - `googleSheetId`
    - `crmApiUrl`
  - **Include other fields:** enabled (preserves webhook data).
- **Input/Output:**
  - **Input:** Data from webhook.
  - **Output:** Same data plus configuration fields.
  - **Next node:** `Normalize Client Data`
- **Edge cases / failures:**
  - Leaving placeholders unchanged will break downstream nodes (Slack/Sheets/CRM).
  - If you later reference this node by name (the workflow does), renaming it requires updating expressions like `$('Workflow Configuration')...`.

---

### Block 2 — Data Normalization
**Overview:** Converts various possible field names from the incoming form into a consistent schema used across the workflow.  
**Nodes involved:** `Normalize Client Data`

#### Node: Normalize Client Data
- **Type / role:** Set node — schema normalization.
- **Key expressions/variables:**
  - `clientName = {{ $json.body.name || $json.body.clientName || '' }}`
  - `clientEmail = {{ $json.body.email || $json.body.clientEmail || '' }}`
  - `projectType = {{ $json.body.projectType || $json.body.project_type || '' }}`
  - `company = {{ $json.body.company || '' }}`
  - `phone = {{ $json.body.phone || '' }}`
  - **Include other fields:** enabled.
- **Input/Output:**
  - **Input:** Webhook payload + workflow configuration.
  - **Output:** Adds normalized fields used by AI, routing, sheets, Slack, email, calendar, CRM.
  - **Next node:** `Generate Onboarding Checklist`
- **Edge cases / failures:**
  - If `$json.body` is missing (caller sends raw fields not nested under `body`), all fields become empty strings. Consider defensive logic if your webhook sends a different structure.
  - `projectType` empty → routing won’t match any rule; downstream steps after routing will not run.

---

### Block 3 — AI Generation
**Overview:** Uses GPT‑4o‑mini to generate structured onboarding content based on the client + project type.  
**Nodes involved:** `Generate Onboarding Checklist`

#### Node: Generate Onboarding Checklist
- **Type / role:** OpenAI (LangChain) node — generates onboarding JSON.
- **Model:** `gpt-4o-mini`
- **Prompt strategy (interpreted):**
  - System-style **instructions** define required JSON structure:
    - `checklist` (5–8 items)
    - `firstTasks` (3–5 items)
    - `deliverables` (milestones)
  - Tailoring guidance based on project type (Strategy/Management/IT).
  - User content passed includes:
    - Client Name, Project Type, Company
- **Output/Connections:**
  - **Input:** Normalized fields from previous node.
  - **Output:** A message-like structure; later nodes reference `{{$json.message.content}}` which should contain a JSON string.
  - **Next node:** `Route by Project Type`
- **Credentials:** OpenAI API credentials (named “n8n free OpenAI API credits” in the JSON).
- **Edge cases / failures:**
  - Model may return non-JSON or invalid JSON → downstream `JSON.parse($json.message.content)` in Slack/email will throw and fail the execution.
  - Rate limits / quota exhaustion → OpenAI node fails.
  - Prompt injection via user-provided fields could degrade structure; enforce JSON strictly (see reproduction section improvements).

---

### Block 4 — Routing + Logging + Internal Notifications
**Overview:** Routes by `projectType`, writes the client record into the correct Google Sheets tab, then notifies the appropriate Slack channel with checklist/tasks.  
**Nodes involved:** `Route by Project Type`, `Log to Google Sheets - Strategy`, `Log to Google Sheets - Management`, `Log to Google Sheets - IT`, `Notify Strategy Consultant`, `Notify Management Consultant`, `Notify IT Consultant`

#### Node: Route by Project Type
- **Type / role:** Switch node — 3-way branch by string equality.
- **Rules (outputs are renamed):**
  - If `{{$json.projectType}} == "Strategy"` → Strategy output
  - If `... == "Management"` → Management output
  - If `... == "IT"` → IT output
- **Connections:**
  - Input from `Generate Onboarding Checklist`
  - Outputs to the three Sheets nodes.
- **Edge cases / failures:**
  - Case sensitivity is effectively enforced by exact match; “strategy” or “IT Consulting” won’t match.
  - No default/fallback route configured → unmatched values result in no downstream actions.

#### Node: Log to Google Sheets - Strategy
- **Type / role:** Google Sheets — append or update a row.
- **Operation:** Append or Update
- **Target sheet:** `Strategy Clients`
- **Document ID:** from config: `$('Workflow Configuration').first().json.googleSheetId`
- **Matching column:** `Email` (used to decide update vs append)
- **Mapping mode:** auto-map input data.
- **Connections:** `Route by Project Type` → this node → `Notify Strategy Consultant`
- **Edge cases / failures:**
  - If the sheet does not have an `Email` column header exactly matching “Email”, matching fails (may append duplicates or error depending on node behavior).
  - Auto-mapping relies on column names matching keys in JSON (e.g., `clientEmail` vs `Email`). With auto-map, you often need explicit mappings or ensure your input keys match the sheet headers.
  - OAuth token expiration / permission issues.

#### Node: Log to Google Sheets - Management
- **Type / role:** Google Sheets — append or update a row.
- **Target sheet:** `Management Clients`
- **Same behavior/risks as Strategy**, with the same `Email` matching requirement.
- **Connections:** `Route by Project Type` → this node → `Notify Management Consultant`

#### Node: Log to Google Sheets - IT
- **Type / role:** Google Sheets — append or update a row.
- **Target sheet:** `IT Clients`
- **Same behavior/risks as above.**
- **Connections:** `Route by Project Type` → this node → `Notify IT Consultant`

#### Node: Notify Strategy Consultant
- **Type / role:** Slack — posts an internal assignment message to a channel.
- **Channel:** from config `strategyConsultantChannel`
- **Message content highlights:**
  - Includes client details and renders checklist + firstTasks by parsing AI JSON:
    - `JSON.parse($json.message.content).checklist ...`
    - `JSON.parse($json.message.content).firstTasks ...`
  - Fallback text if `$json.message.content` is missing: “See full details in Google Sheets”
- **Connections:** from `Log to Google Sheets - Strategy` → to `Send Welcome Email`
- **Edge cases / failures:**
  - If `message.content` exists but is invalid JSON → Slack node fails due to expression error.
  - Missing/invalid Slack channel ID → Slack API error.
  - Slack app missing `chat:write` permission or not a member of the channel.

#### Node: Notify Management Consultant
- **Type / role:** Slack — posts internal assignment message.
- **Channel:** from config `managementConsultantChannel`
- **Connections:** from `Log to Google Sheets - Management` → to `Send Welcome Email`
- **Same JSON.parse risks** as Strategy.

#### Node: Notify IT Consultant
- **Type / role:** Slack — posts internal assignment message.
- **Channel:** from config `itConsultantChannel`
- **Connections:** from `Log to Google Sheets - IT` → to `Send Welcome Email`
- **Same JSON.parse risks** as others.

---

### Block 5 — Client Communication + Scheduling + CRM Sync
**Overview:** Sends a welcome email to the client with the AI-generated checklist/tasks, schedules a kickoff meeting in Google Calendar, then posts the record to the CRM endpoint.  
**Nodes involved:** `Send Welcome Email`, `Schedule Kickoff Meeting`, `Sync to CRM`

#### Node: Send Welcome Email
- **Type / role:** Email Send — outbound email to client.
- **Key configuration:**
  - **To:** `{{$json.clientEmail}}`
  - **From:** placeholder (`<__PLACEHOLDER_VALUE__Your company email address__>`)
  - **Subject:** `Welcome to Our Consulting Services - {{$json.clientName}}`
  - **HTML body:** embeds checklist/tasks by parsing `$json.message.content` as JSON, else shows fallback list items.
- **Connections:** from any Slack notify node → to `Schedule Kickoff Meeting`
- **Edge cases / failures:**
  - Invalid `clientEmail` → email send fails or bounces.
  - `JSON.parse` risk same as Slack.
  - Email node requires SMTP or configured email credentials depending on your n8n setup (not shown in JSON; must be configured in instance).

#### Node: Schedule Kickoff Meeting
- **Type / role:** Google Calendar — creates an event.
- **Timing expressions:**
  - **Start:** `{{$now.plus(2, 'days').set({ hour: 10, minute: 0 }).toISO()}}`
  - **End:** `{{$now.plus(2, 'days').set({ hour: 11, minute: 0 }).toISO()}}`
- **Calendar:** `primary`
- **Additional fields:**
  - Summary: `Kickoff Meeting - {{$json.clientName}}`
  - Attendees: `{{$json.clientEmail}}`
  - Description: includes project type, client name, company, plus agenda.
- **Connections:** `Send Welcome Email` → this node → `Sync to CRM`
- **Edge cases / failures:**
  - Timezone behavior depends on n8n/server and Google Calendar defaults; `toISO()` uses the runtime timezone.
  - If attendee email is invalid, event creation may still succeed but attendee invite can fail.
  - OAuth permission issues (`calendar.events` scope).

#### Node: Sync to CRM
- **Type / role:** HTTP Request — posts to an external CRM endpoint.
- **URL:** `{{$('Workflow Configuration').first().json.crmApiUrl}}`
- **Method:** POST
- **Body type:** JSON
- **Body fields:**
  - `name`, `email`, `company`, `phone`, `projectType`
  - `status`: `"onboarding"`
  - `onboardingData`: `{{ $json.message.content || "{}" }}`
    - Note: as written, this inserts the raw string from the AI response if present.
- **Authentication:** “predefinedCredentialType” (a credential must be selected in n8n UI; not embedded here).
- **Connections:** from `Schedule Kickoff Meeting` (last functional node).
- **Edge cases / failures:**
  - If `onboardingData` is a string containing JSON, your CRM may expect an object; you may need to parse it first.
  - Network errors/timeouts, non-2xx responses.
  - Missing credentials or wrong auth type.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Client Intake Form | Webhook | Entry point: receive client intake submission | — | Workflow Configuration | ## Webhook & Config |
| Workflow Configuration | Set | Store channel IDs, Sheet ID, CRM URL (placeholders) | New Client Intake Form | Normalize Client Data | ## Webhook & Config |
| Normalize Client Data | Set | Normalize inbound fields into consistent schema | Workflow Configuration | Generate Onboarding Checklist | ## Webhook & Config |
| Generate Onboarding Checklist | OpenAI (LangChain) | Generate onboarding checklist/tasks/deliverables JSON | Normalize Client Data | Route by Project Type | ## AI Routing |
| Route by Project Type | Switch | Route execution by `projectType` | Generate Onboarding Checklist | Log to Google Sheets - Strategy; Log to Google Sheets - Management; Log to Google Sheets - IT | ## AI Routing |
| Log to Google Sheets - Strategy | Google Sheets | Append/update Strategy client record | Route by Project Type (Strategy) | Notify Strategy Consultant | ## Log & Notify |
| Log to Google Sheets - Management | Google Sheets | Append/update Management client record | Route by Project Type (Management) | Notify Management Consultant | ## Log & Notify |
| Log to Google Sheets - IT | Google Sheets | Append/update IT client record | Route by Project Type (IT) | Notify IT Consultant | ## Log & Notify |
| Notify Strategy Consultant | Slack | Notify Strategy channel with parsed AI output | Log to Google Sheets - Strategy | Send Welcome Email | ## Log & Notify |
| Notify Management Consultant | Slack | Notify Management channel with parsed AI output | Log to Google Sheets - Management | Send Welcome Email | ## Log & Notify |
| Notify IT Consultant | Slack | Notify IT channel with parsed AI output | Log to Google Sheets - IT | Send Welcome Email | ## Log & Notify |
| Send Welcome Email | Email Send | Send client welcome email + checklist/tasks | Notify Strategy Consultant; Notify Management Consultant; Notify IT Consultant | Schedule Kickoff Meeting | ## CRM |
| Schedule Kickoff Meeting | Google Calendar | Create kickoff event (+2 days, 10–11) | Send Welcome Email | Sync to CRM | ## CRM |
| Sync to CRM | HTTP Request | POST onboarding record to CRM endpoint | Schedule Kickoff Meeting | — | ## CRM |
| Sticky Note | Sticky Note | Comment block | — | — |  |
| Sticky Note1 | Sticky Note | Comment block | — | — |  |
| Sticky Note2 | Sticky Note | Comment block | — | — |  |
| Sticky Note3 | Sticky Note | Comment block | — | — |  |
| Sticky Note4 | Sticky Note | Comment block | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Automated Client Onboarding and Task Assignment System**
   - (Optional) Set execution order to default (`v1`), matching the original.

2. **Add Webhook node**
   - Node: **Webhook**
   - Name: `New Client Intake Form`
   - Method: **POST**
   - Path: **client-intake**
   - Response: **Last node**
   - Save and copy the Test/Production webhook URL for your intake form.

3. **Add Set node for configuration**
   - Node: **Set**
   - Name: `Workflow Configuration`
   - Turn on **Include Other Fields**
   - Add string fields:
     - `strategyConsultantChannel` = your Slack channel ID for Strategy
     - `managementConsultantChannel` = your Slack channel ID for Management
     - `itConsultantChannel` = your Slack channel ID for IT
     - `googleSheetId` = your Spreadsheet ID
     - `crmApiUrl` = your CRM endpoint URL
   - Connect: `New Client Intake Form` → `Workflow Configuration`

4. **Add Set node to normalize client data**
   - Node: **Set**
   - Name: `Normalize Client Data`
   - Turn on **Include Other Fields**
   - Add fields with expressions:
     - `clientName` = `{{ $json.body.name || $json.body.clientName || '' }}`
     - `clientEmail` = `{{ $json.body.email || $json.body.clientEmail || '' }}`
     - `projectType` = `{{ $json.body.projectType || $json.body.project_type || '' }}`
     - `company` = `{{ $json.body.company || '' }}`
     - `phone` = `{{ $json.body.phone || '' }}`
   - Connect: `Workflow Configuration` → `Normalize Client Data`

5. **Add OpenAI node (LangChain)**
   - Node: **OpenAI (LangChain)** (the `@n8n/n8n-nodes-langchain.openAi` node)
   - Name: `Generate Onboarding Checklist`
   - Credentials: configure/select an **OpenAI API** credential
   - Model: **gpt-4o-mini**
   - Instructions: require JSON with keys `checklist`, `firstTasks`, `deliverables`, and include the tailoring rules for Strategy/Management/IT.
   - User content (message) should include:
     - `Client Name: {{ $json.clientName }}`
     - `Project Type: {{ $json.projectType }}`
     - `Company: {{ $json.company }}`
   - Connect: `Normalize Client Data` → `Generate Onboarding Checklist`

6. **Add Switch node for routing**
   - Node: **Switch**
   - Name: `Route by Project Type`
   - Add 3 rules (string equals):
     - `{{ $json.projectType }}` equals `Strategy` → output name “Strategy”
     - equals `Management` → output name “Management”
     - equals `IT` → output name “IT”
   - Connect: `Generate Onboarding Checklist` → `Route by Project Type`

7. **Add Google Sheets nodes (3 branches)**
   - Credentials: configure **Google Sheets OAuth2**.
   - For each branch, create a **Google Sheets** node:
     1) Name: `Log to Google Sheets - Strategy`
        - Operation: **Append or Update**
        - Document ID: `{{ $('Workflow Configuration').first().json.googleSheetId }}`
        - Sheet name: `Strategy Clients`
        - Matching columns: `Email`
        - Mapping: Auto-map
        - Connect: `Route by Project Type (Strategy output)` → this node
     2) Name: `Log to Google Sheets - Management`
        - Same settings, sheet name: `Management Clients`
        - Connect: `Route by Project Type (Management output)` → this node
     3) Name: `Log to Google Sheets - IT`
        - Same settings, sheet name: `IT Clients`
        - Connect: `Route by Project Type (IT output)` → this node

8. **Add Slack notification nodes (3 branches)**
   - Credentials: configure Slack (OAuth or bot token, depending on your n8n Slack node setup).
   - For each branch, add a **Slack** node configured to post to a channel:
     1) Name: `Notify Strategy Consultant`
        - Send message to **channel**
        - Channel ID: `{{ $('Workflow Configuration').first().json.strategyConsultantChannel }}`
        - Text includes client info and parses AI JSON from `{{$json.message.content}}` to render checklist and first tasks.
        - Connect: `Log to Google Sheets - Strategy` → `Notify Strategy Consultant`
     2) Name: `Notify Management Consultant`
        - Channel ID: `{{ $('Workflow Configuration').first().json.managementConsultantChannel }}`
        - Connect: `Log to Google Sheets - Management` → `Notify Management Consultant`
     3) Name: `Notify IT Consultant`
        - Channel ID: `{{ $('Workflow Configuration').first().json.itConsultantChannel }}`
        - Connect: `Log to Google Sheets - IT` → `Notify IT Consultant`

9. **Add Email Send node**
   - Node: **Email Send**
   - Name: `Send Welcome Email`
   - Configure your email credentials (SMTP or provider integration supported by your n8n instance).
   - From: your company email address
   - To: `{{ $json.clientEmail }}`
   - Subject: `Welcome to Our Consulting Services - {{ $json.clientName }}`
   - HTML body: include checklist and tasks by parsing `{{$json.message.content}}` (with fallback if missing).
   - Connect each Slack node to this single email node:
     - `Notify Strategy Consultant` → `Send Welcome Email`
     - `Notify Management Consultant` → `Send Welcome Email`
     - `Notify IT Consultant` → `Send Welcome Email`

10. **Add Google Calendar node**
   - Node: **Google Calendar**
   - Name: `Schedule Kickoff Meeting`
   - Credentials: configure Google Calendar OAuth2.
   - Calendar: `primary`
   - Start: `{{ $now.plus(2, 'days').set({ hour: 10, minute: 0 }).toISO() }}`
   - End: `{{ $now.plus(2, 'days').set({ hour: 11, minute: 0 }).toISO() }}`
   - Summary: `Kickoff Meeting - {{ $json.clientName }}`
   - Attendees: `{{ $json.clientEmail }}`
   - Description: project overview + agenda
   - Connect: `Send Welcome Email` → `Schedule Kickoff Meeting`

11. **Add HTTP Request node for CRM sync**
   - Node: **HTTP Request**
   - Name: `Sync to CRM`
   - Method: **POST**
   - URL: `{{ $('Workflow Configuration').first().json.crmApiUrl }}`
   - Body: **JSON**, send body enabled
   - Authentication: select the required credential type for your CRM (API key, OAuth2, basic auth, etc.)
   - JSON payload fields: name/email/company/phone/projectType/status/onboardingData as in the workflow
   - Connect: `Schedule Kickoff Meeting` → `Sync to CRM`

12. **(Optional) Add Sticky Notes for documentation**
   - Add sticky notes with these headings around the related nodes:
     - “## Webhook & Config”
     - “## AI Routing”
     - “## Log & Notify”
     - “## CRM”
     - “## Main …” (include setup checklist + author/company/contact)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Main  This workflow automates consulting client onboarding, task creation, and routing. AI generates a checklist based on project type and assigns tasks to the correct team member.  ## Setup  1. Connect your Webhook/Form credentials for client intake. 2. Configure Google Sheets to store client info. 3. Connect OpenAI credentials for checklist/task generation. 4. Connect Slack for internal notifications. 5. Connect email for client communication. 6. Optional: CRM integration for automated logging.  **Author:** Hyrum Hurst, AI Automation Engineer **Company:** QuarterSmart **Contact:** hyrum@quartersmart.com | Sticky note “Main” block |
| ## Webhook & Config | Sticky note section label |
| ## AI Routing | Sticky note section label |
| ## Log & Notify | Sticky note section label |
| ## CRM | Sticky note section label |