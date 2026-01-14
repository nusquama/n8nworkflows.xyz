Sync your HRIS employee directory with Microsoft Teams, Coda, and Slack

https://n8nworkflows.xyz/workflows/sync-your-hris-employee-directory-with-microsoft-teams--coda--and-slack-12586


# Sync your HRIS employee directory with Microsoft Teams, Coda, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name (in JSON):** *Employee Directory Sync – Microsoft Teams & Coda*  
**Provided title:** *Sync your HRIS employee directory with Microsoft Teams, Coda, and Slack*

### Purpose
This workflow runs on a 24-hour schedule to synchronize an HR employee directory into a Coda table and notify stakeholders in Microsoft Teams (and optionally Slack). It retrieves employees from an HR REST API, validates the response, processes employees individually (batched), routes active vs inactive employees, writes active employees to Coda, and posts notifications.

### Target use cases
- Keeping an internal directory (Coda) aligned with an HRIS (HR API).
- Daily visibility in Teams/Slack about additions/updates or inactive employees.
- Early alerting when the HR API is unavailable or returns non-200 responses.

### Logical blocks
1. **1.1 Retrieval & Validation**: Trigger daily run, call HR API, gate downstream processing on successful status code; otherwise alert in Teams.
2. **1.2 Employee List Expansion & Batch Processing**: Convert HR response array into per-employee items and process in batches.
3. **1.3 Normalization & Routing**: Map/normalize employee fields and route by active status.
4. **1.4 Storage to Coda (Active only)**: Prepare a Coda-ready row and create/update the employee record in Coda.
5. **1.5 Notifications (Active & Inactive)**: Send Teams notifications for both paths; Slack notification occurs only after the “active synced” Teams message.

---

## 2. Block-by-Block Analysis

### 2.1 Retrieval & Validation

**Overview:** Triggers every 24 hours, calls the HR API, and checks for a successful HTTP status code. If the API call is not successful, it sends an immediate Teams alert and stops further processing.

**Nodes involved:**
- Daily Employee Sync
- Fetch HR Employees
- Check API Response
- Teams Alert - API Failure

#### Node: Daily Employee Sync
- **Type / role:** Schedule Trigger; starts the workflow on a time interval.
- **Configuration choices:** Runs every **24 hours** (`hoursInterval: 24`).
- **Key expressions/variables:** None.
- **Inputs/outputs:** Entry node → outputs to **Fetch HR Employees**.
- **Version requirements:** `typeVersion 1.1` (standard schedule node behavior).
- **Edge cases / failures:**
  - Missed executions if n8n instance is down.
  - Timezone considerations (interval-based rather than cron-based schedule).

#### Node: Fetch HR Employees
- **Type / role:** HTTP Request; retrieves employee data from HR system.
- **Configuration choices:**
  - URL: `https://api.your-hr-system.com/employees`
  - No explicit auth configured in the JSON; expected to be added (headers/OAuth/API key, etc.).
- **Key expressions/variables:** None.
- **Inputs/outputs:** Input from **Daily Employee Sync** → output to **Check API Response**.
- **Version requirements:** `typeVersion 4.2` (HTTP node v4 line).
- **Edge cases / failures:**
  - Auth failures (401/403) if credentials not configured.
  - Non-JSON responses or unexpected payload shape (breaks downstream code expecting `body.employees`).
  - Timeouts / rate limits (429) / transient 5xx responses.

#### Node: Check API Response
- **Type / role:** IF; validates API response status code before processing.
- **Configuration choices:**
  - Condition: numeric compare `={{ $json.statusCode || 200 }}` **equals** `200`.
  - This design defaults missing `statusCode` to `200`, allowing processing even if the node output does not include `statusCode`.
- **Key expressions/variables:**
  - `{{ $json.statusCode || 200 }}`
- **Inputs/outputs:**
  - Input from **Fetch HR Employees**
  - **True output** → **Expand Employee List**
  - **False output** → **Teams Alert - API Failure**
- **Version requirements:** `typeVersion 2`.
- **Edge cases / failures:**
  - If HTTP node does not populate `statusCode`, the `|| 200` fallback may allow bad payloads through; downstream code will then treat missing `body.employees` as empty list.
  - If HR API returns 200 but error inside body (e.g., `{ error: ... }`), this gate will not catch it.

#### Node: Teams Alert - API Failure
- **Type / role:** Microsoft Teams; sends an alert message when the HR API retrieval fails.
- **Configuration choices:**
  - Resource: `chatMessage`
  - Content type: `html`
  - Message: `<b>Employee sync failed</b><br/>Unable to fetch employee list from HR system.`
  - `chatId` is configured in “list” mode but currently empty in JSON (must be set).
- **Inputs/outputs:** Input from **Check API Response (false)**; no downstream nodes.
- **Version requirements:** `typeVersion 2`.
- **Edge cases / failures:**
  - Missing/invalid Teams OAuth credentials.
  - Missing chat/channel selection (chatId empty) causing runtime failure.
  - Permissions issues posting to the selected chat/channel.

---

### 2.2 Employee List Expansion & Batch Processing

**Overview:** Converts the HR response array into separate items (one per employee) and processes them in batches to reduce rate-limit risks for Coda/Teams/Slack.

**Nodes involved:**
- Expand Employee List
- Split Employees

#### Node: Expand Employee List
- **Type / role:** Code; reshapes API response into item-per-employee.
- **Configuration choices (interpreted):**
  - Reads `items[0].json.body?.employees || []`
  - Returns `employees.map(e => ({ json: e }))`
- **Key expressions/variables:**
  - Uses optional chaining on `body?.employees`.
- **Inputs/outputs:** Input from **Check API Response (true)** → output to **Split Employees**.
- **Version requirements:** `typeVersion 2` (Code node).
- **Edge cases / failures:**
  - If the HTTP node’s response body is not under `.body` (common differences depending on HTTP node settings), this will produce an empty employee list (silent “success” but no processing).
  - If `employees` is not an array, `.map` will throw.

#### Node: Split Employees
- **Type / role:** Split In Batches; processes employees incrementally to control throughput.
- **Configuration choices:** Options empty; **batch size not explicitly set** (n8n default applies).
- **Key expressions/variables:** None.
- **Inputs/outputs:** Input from **Expand Employee List** → output to **Map Employee Fields**.
- **Version requirements:** `typeVersion 3`.
- **Edge cases / failures:**
  - Default batch size may be too large or too small depending on rate limits.
  - Without an explicit loop-back connection (common pattern), behavior depends on downstream design; in this workflow JSON, it proceeds forward only (no “continue” connection shown).

---

### 2.3 Normalization & Routing

**Overview:** Normalizes each employee record into a consistent schema and routes employees based on active status. Active employees go to Coda + notifications; inactive employees only trigger an “inactive” Teams notification.

**Nodes involved:**
- Map Employee Fields
- Is Active Employee?

#### Node: Map Employee Fields
- **Type / role:** Code; normalizes schema and creates helper fields for messaging/routing.
- **Configuration choices (interpreted mapping):**
  - Builds:
    - `employeeId`: `emp.id || emp.employeeId`
    - `name`: `${emp.firstName} ${emp.lastName}` trimmed
    - `email`: `emp.email`
    - `department`: `emp.department || 'Unknown'`
    - `status`: `emp.status || 'inactive'`
    - `active`: boolean derived from `(emp.status || '').toLowerCase() === 'active'`
    - `slackHandle`: `emp.slackHandle || ''`
    - `teamsMessage`: initialized empty
    - `slackMessage`: initialized empty
- **Key expressions/variables:** Derived `active` boolean from status string.
- **Inputs/outputs:** Input from **Split Employees** → output to **Is Active Employee?**.
- **Version requirements:** `typeVersion 2`.
- **Edge cases / failures:**
  - Missing `firstName/lastName/email` leads to partial/blank fields.
  - If HR schema differs significantly (nested fields), mapping must be adapted.
  - Non-string `status` could break `.toLowerCase()` if not handled (currently `emp.status || ''` mitigates null/undefined).

#### Node: Is Active Employee?
- **Type / role:** IF; routes employees based on `active` boolean.
- **Configuration choices:**
  - Condition: `={{ $json.active }}` **is true**
- **Inputs/outputs:**
  - Input from **Map Employee Fields**
  - **True output (active)** → **Prepare Coda Row**
  - **False output (inactive)** → **Set Teams Message (Inactive)**
- **Version requirements:** `typeVersion 2`.
- **Edge cases / failures:**
  - If `active` is missing/non-boolean, routing may be incorrect.
  - “Active” detection is status-string based; HR systems often have more statuses (leave, terminated, onboarding).

---

### 2.4 Storage to Coda (Active only)

**Overview:** Prepares a row payload and writes the employee to a Coda table. The Coda node is configured with a document ID and table name, but the field mapping within the “Prepare Coda Row” Set node is not defined in the JSON (must be configured).

**Nodes involved:**
- Prepare Coda Row
- Create/Update Employee in Coda

#### Node: Prepare Coda Row
- **Type / role:** Set; intended to transform normalized employee fields into a structure expected by the Coda node.
- **Configuration choices:** `options: {}`; no explicit fields shown in JSON (likely incomplete placeholder).
- **Inputs/outputs:** Input from **Is Active Employee? (true)** → output to **Create/Update Employee in Coda**.
- **Version requirements:** `typeVersion 3.4`.
- **Edge cases / failures:**
  - As configured (no fields set), it may pass data through unchanged, which may not match what the Coda node expects.
  - If it is intended to rename keys to match Coda columns, missing configuration will cause incorrect writes.

#### Node: Create/Update Employee in Coda
- **Type / role:** Coda; writes employee data into a Coda doc table.
- **Configuration choices:**
  - Doc ID: `YOUR_CODA_DOC_ID` (must be replaced)
  - Table: `Employees`
  - Operation is not explicitly shown; node name implies upsert, but the parameters do not show which action (depends on node defaults and UI configuration not captured here).
- **Key expressions/variables:** None shown; likely expects incoming JSON keys to map to Coda columns.
- **Inputs/outputs:** Input from **Prepare Coda Row** → output to **Set Teams Message (Active)**.
- **Version requirements:** `typeVersion 1`.
- **Edge cases / failures:**
  - Invalid Coda API token / doc permissions.
  - Table name mismatch (“Employees” must exist).
  - If upsert requires a key column (e.g., Employee ID) and it’s not set, duplicates may occur.

---

### 2.5 Notifications (Active & Inactive)

**Overview:** Builds and sends Teams notifications for active and inactive employees. For active employees, it also posts to Slack after the Teams message. The Set nodes currently do not define how `teamsMessage` is populated (must be configured).

**Nodes involved:**
- Set Teams Message (Active)
- Teams Notify - Employee Synced
- Slack Notify - New Employee
- Set Teams Message (Inactive)
- Teams Notify - Employee Inactive

#### Node: Set Teams Message (Active)
- **Type / role:** Set; intended to populate `teamsMessage` for active employee notification.
- **Configuration choices:** `options: {}`; no explicit value assignments shown in JSON.
- **Inputs/outputs:** Input from **Create/Update Employee in Coda** → output to **Teams Notify - Employee Synced**.
- **Version requirements:** `typeVersion 3.4`.
- **Edge cases / failures:**
  - If `teamsMessage` remains empty, Teams will receive a blank message (or API may reject it).

#### Node: Teams Notify - Employee Synced
- **Type / role:** Microsoft Teams; posts an HTML message about the synced employee.
- **Configuration choices:**
  - Resource: `chatMessage`
  - Content type: `html`
  - Message: `={{ $json.teamsMessage }}`
  - `chatId` in list mode but empty in JSON (must be set).
- **Inputs/outputs:** Input from **Set Teams Message (Active)** → output to **Slack Notify - New Employee**.
- **Version requirements:** `typeVersion 2`.
- **Edge cases / failures:**
  - Empty chatId or invalid Teams credential.
  - Message HTML issues (Teams has HTML limitations).

#### Node: Slack Notify - New Employee
- **Type / role:** Slack; posts a message (operation: `postMessage`).
- **Configuration choices:**
  - Operation: `postMessage`
  - Channel/text not shown in JSON (requires configuration in node UI/params).
- **Inputs/outputs:** Input from **Teams Notify - Employee Synced**; no downstream node.
- **Version requirements:** `typeVersion 2.1`.
- **Edge cases / failures:**
  - Missing Slack credential or missing channel.
  - Rate limits if many employees are processed.
  - If the intent is “new employee only”, current workflow has no dedup/new-detection logic; it will post for every active employee processed unless configured otherwise.

#### Node: Set Teams Message (Inactive)
- **Type / role:** Set; intended to populate `teamsMessage` for inactive employees.
- **Configuration choices:** `options: {}`; no explicit fields shown.
- **Inputs/outputs:** Input from **Is Active Employee? (false)** → output to **Teams Notify - Employee Inactive**.
- **Version requirements:** `typeVersion 3.4`.
- **Edge cases / failures:**
  - Blank `teamsMessage` causing empty Teams notification.

#### Node: Teams Notify - Employee Inactive
- **Type / role:** Microsoft Teams; posts an HTML message about an inactive employee.
- **Configuration choices:**
  - Resource: `chatMessage`
  - Content type: `html`
  - Message: `={{ $json.teamsMessage }}`
  - `chatId` in list mode but empty in JSON (must be set).
- **Inputs/outputs:** Input from **Set Teams Message (Inactive)**; no downstream node.
- **Version requirements:** `typeVersion 2`.
- **Edge cases / failures:**
  - Same Teams credential/chat configuration concerns as the other Teams nodes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Employee Sync | Schedule Trigger | Starts workflow every 24 hours | — | Fetch HR Employees | ## Data Retrieval & Validation<br/>This section contains the trigger, data-fetch, and validation logic. The *Daily Employee Sync* schedule node fires every 24 hours (adjustable). It immediately calls the HR system’s REST API via the *Fetch HR Employees* HTTP Request node, expecting a structured JSON payload. The subsequent *Check API Response* IF node safeguards the rest of the workflow by confirming that the HTTP status code is 200. If the call fails, an instant Microsoft Teams alert is generated so stakeholders can resolve the issue quickly. By isolating retrieval and validation here, downstream nodes only receive reliable data, reducing unnecessary error handling in later stages. |
| Fetch HR Employees | HTTP Request | Pull employee list from HR API | Daily Employee Sync | Check API Response | ## Data Retrieval & Validation<br/>This section contains the trigger, data-fetch, and validation logic. The *Daily Employee Sync* schedule node fires every 24 hours (adjustable). It immediately calls the HR system’s REST API via the *Fetch HR Employees* HTTP Request node, expecting a structured JSON payload. The subsequent *Check API Response* IF node safeguards the rest of the workflow by confirming that the HTTP status code is 200. If the call fails, an instant Microsoft Teams alert is generated so stakeholders can resolve the issue quickly. By isolating retrieval and validation here, downstream nodes only receive reliable data, reducing unnecessary error handling in later stages. |
| Check API Response | IF | Gate processing on HTTP status | Fetch HR Employees | Expand Employee List; Teams Alert - API Failure | ## Data Retrieval & Validation<br/>This section contains the trigger, data-fetch, and validation logic. The *Daily Employee Sync* schedule node fires every 24 hours (adjustable). It immediately calls the HR system’s REST API via the *Fetch HR Employees* HTTP Request node, expecting a structured JSON payload. The subsequent *Check API Response* IF node safeguards the rest of the workflow by confirming that the HTTP status code is 200. If the call fails, an instant Microsoft Teams alert is generated so stakeholders can resolve the issue quickly. By isolating retrieval and validation here, downstream nodes only receive reliable data, reducing unnecessary error handling in later stages. |
| Teams Alert - API Failure | Microsoft Teams | Alert stakeholders when HR API fetch fails | Check API Response (false) | — | ## Data Retrieval & Validation<br/>This section contains the trigger, data-fetch, and validation logic. The *Daily Employee Sync* schedule node fires every 24 hours (adjustable). It immediately calls the HR system’s REST API via the *Fetch HR Employees* HTTP Request node, expecting a structured JSON payload. The subsequent *Check API Response* IF node safeguards the rest of the workflow by confirming that the HTTP status code is 200. If the call fails, an instant Microsoft Teams alert is generated so stakeholders can resolve the issue quickly. By isolating retrieval and validation here, downstream nodes only receive reliable data, reducing unnecessary error handling in later stages. |
| Expand Employee List | Code | Split HR response into one item per employee | Check API Response (true) | Split Employees | ## Employee Processing<br/>After the raw list arrives, *Expand Employee List* converts it into one-item-per-employee format. The *Split Employees* node ensures batched processing to respect rate limits on both Coda and notification services. The *Map Employee Fields* code node normalizes data so it can be consumed consistently—adding derived fields like the `active` boolean. The *Is Active Employee?* IF node then routes records: active staff proceed to storage and announcements, while inactive individuals merely trigger status notifications. This modular design keeps the processing logic clean, making it easy to add further enrichment steps (e.g., role mapping, manager look-ups) without altering other workflow parts. |
| Split Employees | Split In Batches | Batch employees for rate limit control | Expand Employee List | Map Employee Fields | ## Employee Processing<br/>After the raw list arrives, *Expand Employee List* converts it into one-item-per-employee format. The *Split Employees* node ensures batched processing to respect rate limits on both Coda and notification services. The *Map Employee Fields* code node normalizes data so it can be consumed consistently—adding derived fields like the `active` boolean. The *Is Active Employee?* IF node then routes records: active staff proceed to storage and announcements, while inactive individuals merely trigger status notifications. This modular design keeps the processing logic clean, making it easy to add further enrichment steps (e.g., role mapping, manager look-ups) without altering other workflow parts. |
| Map Employee Fields | Code | Normalize HR schema and compute `active` flag | Split Employees | Is Active Employee? | ## Employee Processing<br/>After the raw list arrives, *Expand Employee List* converts it into one-item-per-employee format. The *Split Employees* node ensures batched processing to respect rate limits on both Coda and notification services. The *Map Employee Fields* code node normalizes data so it can be consumed consistently—adding derived fields like the `active` boolean. The *Is Active Employee?* IF node then routes records: active staff proceed to storage and announcements, while inactive individuals merely trigger status notifications. This modular design keeps the processing logic clean, making it easy to add further enrichment steps (e.g., role mapping, manager look-ups) without altering other workflow parts. |
| Is Active Employee? | IF | Route active vs inactive employees | Map Employee Fields | Prepare Coda Row; Set Teams Message (Inactive) | ## Employee Processing<br/>After the raw list arrives, *Expand Employee List* converts it into one-item-per-employee format. The *Split Employees* node ensures batched processing to respect rate limits on both Coda and notification services. The *Map Employee Fields* code node normalizes data so it can be consumed consistently—adding derived fields like the `active` boolean. The *Is Active Employee?* IF node then routes records: active staff proceed to storage and announcements, while inactive individuals merely trigger status notifications. This modular design keeps the processing logic clean, making it easy to add further enrichment steps (e.g., role mapping, manager look-ups) without altering other workflow parts. |
| Prepare Coda Row | Set | Shape data to Coda table schema | Is Active Employee? (true) | Create/Update Employee in Coda | ## Storage & Notifications<br/>Active employee records flow into *Prepare Coda Row*, translating mapped fields into Coda-ready key/value pairs. The *Create/Update Employee in Coda* node writes each employee into the “Employees” table, keeping your directory up to date. Success triggers a Teams message (and an optional Slack post) summarizing the action for transparency. Inactive records produce a distinct Teams notification so HR can archive or reactivate as needed. Centralizing storage and messaging here aids maintainability; if your organization adopts additional channels (e.g., email or database storage), you can append nodes inside this gray block without touching upstream logic. |
| Create/Update Employee in Coda | Coda | Write employee record to Coda table | Prepare Coda Row | Set Teams Message (Active) | ## Storage & Notifications<br/>Active employee records flow into *Prepare Coda Row*, translating mapped fields into Coda-ready key/value pairs. The *Create/Update Employee in Coda* node writes each employee into the “Employees” table, keeping your directory up to date. Success triggers a Teams message (and an optional Slack post) summarizing the action for transparency. Inactive records produce a distinct Teams notification so HR can archive or reactivate as needed. Centralizing storage and messaging here aids maintainability; if your organization adopts additional channels (e.g., email or database storage), you can append nodes inside this gray block without touching upstream logic. |
| Set Teams Message (Active) | Set | Build Teams message for active employee | Create/Update Employee in Coda | Teams Notify - Employee Synced | ## Storage & Notifications<br/>Active employee records flow into *Prepare Coda Row*, translating mapped fields into Coda-ready key/value pairs. The *Create/Update Employee in Coda* node writes each employee into the “Employees” table, keeping your directory up to date. Success triggers a Teams message (and an optional Slack post) summarizing the action for transparency. Inactive records produce a distinct Teams notification so HR can archive or reactivate as needed. Centralizing storage and messaging here aids maintainability; if your organization adopts additional channels (e.g., email or database storage), you can append nodes inside this gray block without touching upstream logic. |
| Teams Notify - Employee Synced | Microsoft Teams | Post success message to Teams | Set Teams Message (Active) | Slack Notify - New Employee | ## Storage & Notifications<br/>Active employee records flow into *Prepare Coda Row*, translating mapped fields into Coda-ready key/value pairs. The *Create/Update Employee in Coda* node writes each employee into the “Employees” table, keeping your directory up to date. Success triggers a Teams message (and an optional Slack post) summarizing the action for transparency. Inactive records produce a distinct Teams notification so HR can archive or reactivate as needed. Centralizing storage and messaging here aids maintainability; if your organization adopts additional channels (e.g., email or database storage), you can append nodes inside this gray block without touching upstream logic. |
| Slack Notify - New Employee | Slack | Post a message to Slack | Teams Notify - Employee Synced | — | ## Storage & Notifications<br/>Active employee records flow into *Prepare Coda Row*, translating mapped fields into Coda-ready key/value pairs. The *Create/Update Employee in Coda* node writes each employee into the “Employees” table, keeping your directory up to date. Success triggers a Teams message (and an optional Slack post) summarizing the action for transparency. Inactive records produce a distinct Teams notification so HR can archive or reactivate as needed. Centralizing storage and messaging here aids maintainability; if your organization adopts additional channels (e.g., email or database storage), you can append nodes inside this gray block without touching upstream logic. |
| Set Teams Message (Inactive) | Set | Build Teams message for inactive employee | Is Active Employee? (false) | Teams Notify - Employee Inactive | ## Storage & Notifications<br/>Active employee records flow into *Prepare Coda Row*, translating mapped fields into Coda-ready key/value pairs. The *Create/Update Employee in Coda* node writes each employee into the “Employees” table, keeping your directory up to date. Success triggers a Teams message (and an optional Slack post) summarizing the action for transparency. Inactive records produce a distinct Teams notification so HR can archive or reactivate as needed. Centralizing storage and messaging here aids maintainability; if your organization adopts additional channels (e.g., email or database storage), you can append nodes inside this gray block without touching upstream logic. |
| Teams Notify - Employee Inactive | Microsoft Teams | Post inactive status message to Teams | Set Teams Message (Inactive) | — | ## Storage & Notifications<br/>Active employee records flow into *Prepare Coda Row*, translating mapped fields into Coda-ready key/value pairs. The *Create/Update Employee in Coda* node writes each employee into the “Employees” table, keeping your directory up to date. Success triggers a Teams message (and an optional Slack post) summarizing the action for transparency. Inactive records produce a distinct Teams notification so HR can archive or reactivate as needed. Centralizing storage and messaging here aids maintainability; if your organization adopts additional channels (e.g., email or database storage), you can append nodes inside this gray block without touching upstream logic. |
| Employee Directory Sync – Overview | Sticky Note | Documentation / operator guidance | — | — | ## How it works<br/>This workflow runs automatically every 24 hours to synchronize employee information between your HR system and your internal directory in Coda. First, it pulls the complete employee list from the HR API and verifies a successful response. Each employee record is processed one-by-one: fields are mapped, active employees are saved to Coda, and notifications are sent to both Microsoft Teams and Slack. If the HR API call fails, a Teams alert is dispatched immediately so the HR or IT team can investigate.<br/><br/>## Setup steps<br/>1. Add your HR API base URL and credentials to the HTTP Request node.<br/>2. Create a Coda doc and table named “Employees” (or update the table name in the Coda node).<br/>3. Supply your Coda API credential and doc ID in the Coda node parameters.<br/>4. Connect a Microsoft Teams OAuth credential and fill in Team & Channel IDs in all Teams nodes.<br/>5. (Optional) Connect a Slack credential and adjust the channel name if desired.<br/>6. Review field mappings in the “Map Employee Fields” and “Prepare Coda Row” nodes to match your schema.<br/>7. Activate the workflow and perform a manual run for validation. |
| Section – Retrieval | Sticky Note | Documentation section label | — | — |  |
| Section – Processing | Sticky Note | Documentation section label | — | — |  |
| Section – Storage & Notify | Sticky Note | Documentation section label | — | — |  |

> Note: Sticky notes “Section – Retrieval/Processing/Storage & Notify” contain the detailed section text already duplicated onto the relevant operational nodes above. The “Employee Directory Sync – Overview” note is kept as-is as global context.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Employee Directory Sync – Microsoft Teams & Coda** (or your preferred name).

2. **Add Trigger: “Schedule Trigger”**
   - Node name: **Daily Employee Sync**
   - Set **Interval** to every **24 hours** (or your desired cadence).

3. **Add “HTTP Request” node**
   - Node name: **Fetch HR Employees**
   - Method: `GET`
   - URL: `https://api.your-hr-system.com/employees` (replace with your HRIS endpoint)
   - Authentication:
     - Add the required mechanism (API key header / Bearer token / OAuth2), depending on your HRIS.
   - Ensure the response is parsed as JSON (default in most setups).

4. **Add “IF” node to validate response**
   - Node name: **Check API Response**
   - Condition (Number):
     - Value 1: `{{ $json.statusCode || 200 }}`
     - Operation: **equal**
     - Value 2: `200`
   - Connect:
     - **Fetch HR Employees → Check API Response**

5. **Add Microsoft Teams node for failure alert**
   - Node name: **Teams Alert - API Failure**
   - Resource: **Chat message**
   - Content type: **HTML**
   - Message:
     - `<b>Employee sync failed</b><br/>Unable to fetch employee list from HR system.`
   - Credentials:
     - Configure a **Microsoft Teams OAuth2** credential in n8n.
   - Select destination:
     - Set the **chat/channel** (fill the chatId/team/channel as required by your node UI).
   - Connect:
     - **Check API Response (false) → Teams Alert - API Failure**

6. **Add “Code” node to expand employees**
   - Node name: **Expand Employee List**
   - Paste code (adjust `body.employees` path to match your HRIS payload):
     - Reads the array and returns one item per employee.
   - Connect:
     - **Check API Response (true) → Expand Employee List**

7. **Add “Split In Batches”**
   - Node name: **Split Employees**
   - Set **Batch Size** explicitly (recommended): e.g., 10–50 depending on API limits.
   - Connect:
     - **Expand Employee List → Split Employees**

8. **Add “Code” node to map/normalize fields**
   - Node name: **Map Employee Fields**
   - Implement mapping to produce:
     - `employeeId`, `name`, `email`, `department`, `status`, `active`, `slackHandle`, plus message placeholders.
   - Connect:
     - **Split Employees → Map Employee Fields**

9. **Add “IF” node for active routing**
   - Node name: **Is Active Employee?**
   - Condition (Boolean):
     - Value 1: `{{ $json.active }}`
     - Operation: **is true**
   - Connect:
     - **Map Employee Fields → Is Active Employee?**

10. **Active path: Add “Set” node to prepare Coda row**
    - Node name: **Prepare Coda Row**
    - Configure fields to match your Coda table columns. Typical approaches:
      - Keep keys aligned to column names (e.g., `Employee ID`, `Name`, `Email`, `Department`, `Status`)
      - Or build the specific structure required by your chosen Coda operation.
    - Connect:
      - **Is Active Employee? (true) → Prepare Coda Row**

11. **Active path: Add “Coda” node**
    - Node name: **Create/Update Employee in Coda**
    - Credentials:
      - Configure **Coda API** credential (token).
    - Parameters:
      - Doc ID: set your Coda document ID
      - Table: `Employees` (or your table name)
    - Choose the appropriate action:
      - If you need upsert behavior, configure matching on a unique key (commonly Employee ID).
    - Connect:
      - **Prepare Coda Row → Create/Update Employee in Coda**

12. **Active path: Add “Set” node for Teams message**
    - Node name: **Set Teams Message (Active)**
    - Set `teamsMessage` to an HTML string using expressions, e.g. include name/department/email.
    - Connect:
      - **Create/Update Employee in Coda → Set Teams Message (Active)**

13. **Active path: Add Microsoft Teams node for success**
    - Node name: **Teams Notify - Employee Synced**
    - Resource: **Chat message**
    - Content type: **HTML**
    - Message: `{{ $json.teamsMessage }}`
    - Configure destination chat/channel.
    - Connect:
      - **Set Teams Message (Active) → Teams Notify - Employee Synced**

14. **Active path (optional): Add Slack node**
    - Node name: **Slack Notify - New Employee**
    - Operation: **Post Message**
    - Configure credentials (Slack OAuth/token in n8n).
    - Set channel and message (optionally use normalized fields).
    - Connect:
      - **Teams Notify - Employee Synced → Slack Notify - New Employee**

15. **Inactive path: Add “Set” node for inactive Teams message**
    - Node name: **Set Teams Message (Inactive)**
    - Set `teamsMessage` to an HTML message describing the inactive employee.
    - Connect:
      - **Is Active Employee? (false) → Set Teams Message (Inactive)**

16. **Inactive path: Add Microsoft Teams node**
    - Node name: **Teams Notify - Employee Inactive**
    - Resource: **Chat message**
    - Content type: **HTML**
    - Message: `{{ $json.teamsMessage }}`
    - Configure destination chat/channel.
    - Connect:
      - **Set Teams Message (Inactive) → Teams Notify - Employee Inactive**

17. **Add sticky notes (optional but recommended)**
    - Create notes for overview and each section to match the intent in the provided workflow.

18. **Validate with a manual execution**
    - Confirm the HR payload path (whether it is `body.employees` or another property).
    - Confirm Coda table columns and mapping.
    - Confirm Teams chat/channel IDs are set.
    - Confirm Slack target channel is set.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow runs automatically every 24 hours to synchronize employee information between your HR system and your internal directory in Coda… plus setup steps 1–7. | From sticky note **Employee Directory Sync – Overview** |
| Data Retrieval & Validation section explanation (trigger → HTTP fetch → status check → Teams alert on failure). | From sticky note **Section – Retrieval** |
| Employee Processing section explanation (expand list → batch → normalize → route active/inactive). | From sticky note **Section – Processing** |
| Storage & Notifications section explanation (prepare Coda row → write to Coda → Teams+Slack; inactive → Teams). | From sticky note **Section – Storage & Notify** |