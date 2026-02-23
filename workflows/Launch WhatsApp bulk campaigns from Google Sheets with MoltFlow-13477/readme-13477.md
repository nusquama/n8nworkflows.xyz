Launch WhatsApp bulk campaigns from Google Sheets with MoltFlow

https://n8nworkflows.xyz/workflows/launch-whatsapp-bulk-campaigns-from-google-sheets-with-moltflow-13477


# Launch WhatsApp bulk campaigns from Google Sheets with MoltFlow

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *WhatsApp Bulk Campaign from Google Sheets via MoltFlow*  
**Purpose:** Launch a personalized WhatsApp bulk campaign by reading contacts from Google Sheets, creating a custom group in MoltFlow, triggering a bulk-send job, waiting briefly, then checking job progress and producing a summary.

**Typical use cases**
- Marketing/outreach campaigns with a contact list maintained in Google Sheets
- Rapid one-off blasts (manual trigger) with standardized message content
- Lightweight monitoring of sending status shortly after launch

### Logical blocks
**1.1 Manual Start & Data Ingestion**
- Trigger workflow manually and read rows from a Google Sheet.

**1.2 Contact Normalization & Campaign Payload**
- Convert sheet rows into MoltFlow “members”, generate a campaign group name, and set default message content.

**1.3 MoltFlow Group Creation**
- Create a MoltFlow custom group from the normalized members list.

**1.4 Bulk Send Job Creation**
- Create a MoltFlow bulk-send job for the group with the selected WhatsApp session and message.

**1.5 Progress Check & Reporting**
- Wait 30 seconds, fetch job status from MoltFlow, and output a concise campaign summary.

---

## 2. Block-by-Block Analysis

### 2.1 Manual Start & Data Ingestion

**Overview:** Starts the workflow on demand and loads contacts from Google Sheets for further processing.

**Nodes involved**
- Start Campaign
- Read Contacts from Sheet

#### Node: Start Campaign
- **Type / role:** Manual Trigger (`manualTrigger`) — entry point.
- **Configuration choices:** No parameters used; runs when user clicks *Execute workflow*.
- **Inputs / outputs:**
  - **Output:** Sends one empty item to **Read Contacts from Sheet**.
- **Edge cases / failures:** None (other than workflow not being executed).
- **Version requirements:** Node typeVersion 1.

#### Node: Read Contacts from Sheet
- **Type / role:** Google Sheets (`googleSheets`) — reads rows from a spreadsheet.
- **Configuration choices (interpreted):**
  - **Operation:** `read`
  - **Document:** provided by **URL** (placeholder `YOUR_GOOGLE_SHEET_URL`)
  - **Sheet name:** `Sheet1` (selected from list)
- **Credentials:** Google Sheets OAuth2 (`googleSheetsOAuth2Api`) named “Google Sheets”.
- **Inputs / outputs:**
  - **Input:** From **Start Campaign**
  - **Output:** One item per row → **Format Contacts**
- **Key fields expected in rows:** `phone`, `name`, optionally variants `Phone`, `Name`. (A `message_override` column is mentioned in sticky note but not implemented in code.)
- **Edge cases / failures:**
  - OAuth not authorized / expired token.
  - Wrong Sheet URL or permissions (403/404).
  - Sheet name mismatch (`Sheet1` not found).
  - Empty sheet → downstream group will be created with 0 members unless filtered out later.
- **Version requirements:** typeVersion 4.5.

---

### 2.2 Contact Normalization & Campaign Payload

**Overview:** Builds a single campaign payload containing session ID, generated group name, normalized phone list, and a default message.

**Nodes involved**
- Format Contacts

#### Node: Format Contacts
- **Type / role:** Code (`code`) — transforms all rows into one campaign object.
- **Configuration choices (interpreted):**
  - **Mode:** `runOnceForAllItems` (important: aggregates all sheet rows into one output item)
  - **Hardcoded variables:**
    - `SESSION_ID = 'YOUR_SESSION_ID'` (must be replaced)
    - `defaultMessage = 'Hi {{name}}, ...'` (contains a `{{name}}` placeholder intended for personalization downstream)
  - **Group naming:** `Campaign YYYY-MM-DD` based on current date.
  - **Phone normalization:** strips non-digits and keeps entries with length ≥ 7.
  - **Member schema produced:** `{ phone: 'digits', name: '...' }`
  - **Output schema:**  
    - `session_id`, `group_name`, `members` (array), `member_count`, `message`
- **Inputs / outputs:**
  - **Input:** Rows from **Read Contacts from Sheet**
  - **Output:** Single campaign item → **Create Custom Group**
- **Key expressions / variables:**
  - Uses `$input.all()` to gather all rows.
  - Handles alternative column casing: `data.phone || data.Phone`, `data.name || data.Name`.
- **Edge cases / failures:**
  - If all phones are invalid/empty → `members` becomes empty; group creation may fail depending on MoltFlow API validation.
  - `SESSION_ID` left as placeholder → bulk send will fail with MoltFlow auth/session error.
  - `{{name}}` is not substituted inside this workflow; personalization depends on MoltFlow’s templating support. If MoltFlow does not replace it, recipients receive literal `{{name}}`.
  - If sheet includes non-string phone values, `String(...)` handles them, but could produce “undefined” if key missing (filtered out by length check).
- **Version requirements:** typeVersion 2.

---

### 2.3 MoltFlow Group Creation

**Overview:** Creates a MoltFlow custom group with the prepared members list.

**Nodes involved**
- Create Custom Group

#### Node: Create Custom Group
- **Type / role:** HTTP Request (`httpRequest`) — calls MoltFlow API to create a custom group.
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://apiv2.waiflow.app/api/v2/custom-groups`
  - **Body type:** JSON (explicitly stringified)
  - **JSON body payload:**
    - `name`: from `$json.group_name`
    - `members`: from `$json.members`
  - **Authentication:** Generic Credential → HTTP Header Auth (`httpHeaderAuth`)
    - Expected header: `X-API-Key: <your_api_key>` (per sticky note)
- **Inputs / outputs:**
  - **Input:** From **Format Contacts**
  - **Output:** Group creation response → **Prepare Bulk Send**
- **Key expressions / variables:**
  - `={{ JSON.stringify({ name: $json.group_name, members: $json.members }) }}`
- **Edge cases / failures:**
  - Invalid/expired API key → 401/403.
  - MoltFlow rejects empty `members` array or invalid phone formats → 400.
  - Large member lists may hit API limits/timeouts.
  - Stringifying JSON manually can cause issues if API expects object (n8n usually accepts object directly). Here it’s forced to string; if MoltFlow expects JSON object, it *still* works because request is sent as JSON, but it’s an extra risk if headers/content-type mismatch.
- **Version requirements:** typeVersion 4.2.

---

### 2.4 Bulk Send Job Creation

**Overview:** Extracts the created group ID and campaign parameters, then creates a MoltFlow bulk-send job.

**Nodes involved**
- Prepare Bulk Send
- Create Bulk Send Job

#### Node: Prepare Bulk Send
- **Type / role:** Code (`code`) — builds bulk-send job payload from prior node outputs.
- **Configuration choices (interpreted):**
  - **Mode:** `runOnceForAllItems`
  - Reads:
    - Group ID from **Create Custom Group** response: `createResponse.id`
    - Session/message from **Format Contacts** via node lookup
  - Sets message type: `text`
- **Inputs / outputs:**
  - **Input:** From **Create Custom Group**
  - **Output:** Single prepared item → **Create Bulk Send Job**
- **Key expressions / variables:**
  - `$('Format Contacts').first().json.session_id`
  - `$('Format Contacts').first().json.message`
  - Fallback for member count: `createResponse.member_count || ...`
- **Edge cases / failures:**
  - If MoltFlow group creation response doesn’t contain `id`, subsequent requests fail.
  - If node name changes (“Format Contacts”), the `$('Format Contacts')...` references break.
  - If multiple items unexpectedly arrive (shouldn’t, but possible if upstream changes), `.first()` may hide data issues.
- **Version requirements:** typeVersion 2.

#### Node: Create Bulk Send Job
- **Type / role:** HTTP Request (`httpRequest`) — triggers the MoltFlow bulk send.
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://apiv2.waiflow.app/api/v2/bulk-send`
  - **Body:** JSON (stringified) containing:
    - `custom_group_id`
    - `session_id`
    - `message_type`
    - `message_content`
  - **Authentication:** HTTP Header Auth (`X-API-Key`) via “MoltFlow API Key” credential
- **Inputs / outputs:**
  - **Input:** From **Prepare Bulk Send**
  - **Output:** Job creation response → **Wait 30s**
- **Key expressions / variables:**
  - `={{ JSON.stringify({ custom_group_id: $json.custom_group_id, session_id: $json.session_id, message_type: $json.message_type, message_content: $json.message_content }) }}`
- **Edge cases / failures:**
  - Invalid session id or disconnected WhatsApp session → API error.
  - Message content length/format limitations (provider-dependent).
  - API rate limiting / 429.
- **Version requirements:** typeVersion 4.2.

---

### 2.5 Progress Check & Reporting

**Overview:** Pauses briefly to let sending start, fetches job progress, and returns a summary object for easy viewing/logging.

**Nodes involved**
- Wait 30s
- Check Campaign Progress
- Campaign Summary

#### Node: Wait 30s
- **Type / role:** Wait (`wait`) — delay before checking job status.
- **Configuration choices:** 30 seconds.
- **Inputs / outputs:**
  - **Input:** From **Create Bulk Send Job**
  - **Output:** Pass-through → **Check Campaign Progress**
- **Edge cases / failures:**
  - None typical; delays workflow completion.
  - If job status takes longer than 30s to update, summary may show early/incomplete values.
- **Version requirements:** typeVersion 1.1.

#### Node: Check Campaign Progress
- **Type / role:** HTTP Request (`httpRequest`) — fetches bulk-send job state.
- **Configuration choices (interpreted):**
  - **Method:** GET
  - **URL:** dynamically built from job id returned by **Create Bulk Send Job**:  
    `https://apiv2.waiflow.app/api/v2/bulk-send/<job_id>`
  - **Authentication:** HTTP Header Auth (`X-API-Key`)
- **Inputs / outputs:**
  - **Input:** From **Wait 30s**
  - **Output:** Job status response → **Campaign Summary**
- **Key expressions / variables:**
  - `={{ 'https://apiv2.waiflow.app/api/v2/bulk-send/' + $('Create Bulk Send Job').first().json.id }}`
- **Edge cases / failures:**
  - If job creation didn’t return `id`, URL becomes invalid.
  - If job ID not found (deleted/expired), 404.
  - Same node-name fragility as above (`$('Create Bulk Send Job')`).
- **Version requirements:** typeVersion 4.2.

#### Node: Campaign Summary
- **Type / role:** Code (`code`) — maps MoltFlow job response into a concise summary.
- **Configuration choices (interpreted):**
  - **Mode:** `runOnceForAllItems`
  - Computes:
    - `remaining = total_recipients - sent_count - failed_count`
  - Outputs key timestamps if present.
- **Inputs / outputs:**
  - **Input:** From **Check Campaign Progress**
  - **Output:** Final workflow output (no downstream nodes)
- **Edge cases / failures:**
  - If any of `total_recipients/sent_count/failed_count` are missing or null, `remaining` can become `NaN`.
  - If API field names differ (e.g., `sent` vs `sent_count`), summary fields will be undefined.
- **Version requirements:** typeVersion 2.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / canvas annotation | — | — | ## WhatsApp Bulk Campaign \nSend personalized WhatsApp campaigns to contacts imported from Google Sheets using [MoltFlow](https://molt.waiflow.app).\n\n**How it works:**\n1. Click Execute to start\n2. Contacts are read from your Google Sheet\n3. A custom group is created in MoltFlow\n4. A bulk send job sends messages with smart delays\n5. Campaign progress is checked and summarized |
| Sticky Note1 | Sticky Note | Setup instructions / canvas annotation | — | — | ## Setup (5 min)\n1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp\n2. Generate API key with Outreach scope preset\n3. Prepare Google Sheet with columns: phone, name, message_override\n4. Connect Google Sheets OAuth2 credential\n5. Paste your Sheet URL in the Google Sheets node\n6. Set YOUR_SESSION_ID in the Format Contacts node\n7. Add MoltFlow API Key: Header Auth with X-API-Key |
| Start Campaign | Manual Trigger | Manual entry point | — | Read Contacts from Sheet |  |
| Read Contacts from Sheet | Google Sheets | Read contacts rows from a spreadsheet | Start Campaign | Format Contacts |  |
| Format Contacts | Code | Normalize contacts and build campaign payload | Read Contacts from Sheet | Create Custom Group |  |
| Create Custom Group | HTTP Request | Create MoltFlow custom group | Format Contacts | Prepare Bulk Send |  |
| Prepare Bulk Send | Code | Assemble payload for bulk send job | Create Custom Group | Create Bulk Send Job |  |
| Create Bulk Send Job | HTTP Request | Start MoltFlow bulk send job | Prepare Bulk Send | Wait 30s |  |
| Wait 30s | Wait | Delay before polling job status | Create Bulk Send Job | Check Campaign Progress |  |
| Check Campaign Progress | HTTP Request | Poll job status from MoltFlow | Wait 30s | Campaign Summary |  |
| Campaign Summary | Code | Produce final summarized status output | Check Campaign Progress | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: “Start Campaign”**
   - Type: **Manual Trigger**
   - No special configuration.
3. **Add node: “Read Contacts from Sheet”**
   - Type: **Google Sheets**
   - Credentials: create/select **Google Sheets OAuth2** credential and authorize access.
   - Operation: **Read**
   - Document: **By URL** → paste your Google Sheet URL.
   - Sheet name: select **Sheet1** (or your actual tab name).
4. **Connect:** Start Campaign → Read Contacts from Sheet.
5. **Add node: “Format Contacts”**
   - Type: **Code**
   - Mode: **Run Once for All Items**
   - Paste logic that:
     - Sets `SESSION_ID` to your MoltFlow/WhatsApp session identifier (replace `YOUR_SESSION_ID`).
     - Defines a default message string.
     - Reads all rows, maps `{phone, name}`, strips non-digits from phone, filters invalid phones.
     - Outputs a single item with: `session_id`, `group_name`, `members[]`, `member_count`, `message`.
6. **Connect:** Read Contacts from Sheet → Format Contacts.
7. **Create credential: “MoltFlow API Key”**
   - Credential type: **HTTP Header Auth**
   - Header name: `X-API-Key`
   - Value: your MoltFlow API key (must include Outreach scope per note).
8. **Add node: “Create Custom Group”**
   - Type: **HTTP Request**
   - Method: **POST**
   - URL: `https://apiv2.waiflow.app/api/v2/custom-groups`
   - Authentication: **Generic Credential Type** → **HTTP Header Auth** → select “MoltFlow API Key”.
   - Body: **JSON**
   - Body fields (conceptually): `{ name: {{$json.group_name}}, members: {{$json.members}} }`
9. **Connect:** Format Contacts → Create Custom Group.
10. **Add node: “Prepare Bulk Send”**
    - Type: **Code**
    - Mode: **Run Once for All Items**
    - Build an item containing:
      - `custom_group_id` from Create Custom Group response (`id`)
      - `session_id` and `message` read from “Format Contacts”
      - `message_type: 'text'`
11. **Connect:** Create Custom Group → Prepare Bulk Send.
12. **Add node: “Create Bulk Send Job”**
    - Type: **HTTP Request**
    - Method: **POST**
    - URL: `https://apiv2.waiflow.app/api/v2/bulk-send`
    - Auth: same **MoltFlow API Key** header auth credential.
    - JSON body: `{ custom_group_id, session_id, message_type, message_content }` (message_content = prepared message).
13. **Connect:** Prepare Bulk Send → Create Bulk Send Job.
14. **Add node: “Wait 30s”**
    - Type: **Wait**
    - Unit: **Seconds**
    - Amount: **30**
15. **Connect:** Create Bulk Send Job → Wait 30s.
16. **Add node: “Check Campaign Progress”**
    - Type: **HTTP Request**
    - Method: **GET**
    - Auth: **MoltFlow API Key**
    - URL: `https://apiv2.waiflow.app/api/v2/bulk-send/{{JOB_ID}}`
      - In n8n expression terms, reference the **Create Bulk Send Job** node’s returned `id`.
17. **Connect:** Wait 30s → Check Campaign Progress.
18. **Add node: “Campaign Summary”**
    - Type: **Code**
    - Mode: **Run Once for All Items**
    - Map the job JSON response into summary fields: job id, status, counts, remaining, timestamps.
19. **Connect:** Check Campaign Progress → Campaign Summary.
20. **(Optional) Add the two Sticky Notes** to document usage and setup:
    - One describing workflow behavior and linking to https://molt.waiflow.app
    - One listing setup steps (API key, sheet columns, session id placement, header auth)

**No sub-workflows are used** (no Execute Workflow node present).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Send personalized WhatsApp campaigns to contacts imported from Google Sheets using MoltFlow. | https://molt.waiflow.app |
| Setup notes: create MoltFlow account, connect WhatsApp, generate API key with Outreach scope preset, prepare sheet columns (phone, name, message_override), connect Google Sheets OAuth2, paste sheet URL, set `YOUR_SESSION_ID`, use header auth `X-API-Key`. | Reflected in workflow sticky note and node credential requirements. |