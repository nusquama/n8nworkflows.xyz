Kick off client projects after Stripe payment with Google Drive, ClickUp, Gmail, Sheets, and Slack

https://n8nworkflows.xyz/workflows/kick-off-client-projects-after-stripe-payment-with-google-drive--clickup--gmail--sheets--and-slack-12567


# Kick off client projects after Stripe payment with Google Drive, ClickUp, Gmail, Sheets, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Payment to Project Kickoff Automation  
**Goal:** When a Stripe payment succeeds, automatically kick off a client project by:
- validating Stripe payload,
- looking up the client in a CRM (Google Sheets),
- creating an “order” record (Google Sheets),
- provisioning a Google Drive folder structure,
- creating a ClickUp project list + standard tasks,
- emailing the client a welcome message with an intake form link,
- notifying the internal team in Slack,
- alerting and stopping on critical failures.

### 1.1 Payment Reception & Configuration
Triggered by Stripe payment events and loads configurable constants (intake form URL, parent Drive folder).

### 1.2 Data Validation & CRM Lookup
Validates required Stripe fields; then searches CRM sheet by customer email. Stops if not found.

### 1.3 Order Logging (Orders Sheet)
Appends an order row with payment + scheduling/status fields.

### 1.4 Project Provisioning (Drive + ClickUp)
Creates a root Drive folder and standard subfolders; creates a ClickUp list and 4 tasks.

### 1.5 Client & Team Communication + Operational Alerts
Sends welcome email, checks send success, notifies Slack; sends Slack alerts on errors (and stops on critical ones).

---

## 2. Block-by-Block Analysis

### Block A — Payment Reception & Base Configuration
**Overview:** Receives Stripe payment events and sets workflow-level constants used later (intake form URL, Drive parent folder).  
**Nodes involved:** `Payment Received`, `Workflow Configuration`

#### Node: Payment Received
- **Type / role:** Stripe Trigger (`n8n-nodes-base.stripeTrigger`) — entry point.
- **Configuration:** Listens to Stripe events:
  - `checkout.session.completed`
  - `invoice.payment_succeeded`
- **Key fields used downstream:**
  - `data.object.customer_details.email`
  - `data.object.customer_details.name`
  - `data.object.metadata.package`
  - `data.object.metadata.lead_id`
  - `data.object.amount_total` / `amount_subtotal`
  - `id` (note: references use both `Payment Received.item.json.id` and `data.object.*`)
- **Connections:**
  - Output → `Workflow Configuration`
- **Credentials:** Stripe API credential (“Stripe account”).
- **Failure modes / edge cases:**
  - Missing `customer_details` (common if not collected in Checkout configuration).
  - Metadata keys (`package`, `lead_id`) not present → later validation fails.
  - Handling two event types: payload structures can differ slightly; ensure expressions are valid for both.
- **Version:** typeVersion 1.

#### Node: Workflow Configuration
- **Type / role:** Set node — central place for editable constants.
- **Configuration choices:**
  - Sets:
    - `intakeFormUrl` = `https://tally.so/r/YOUR_FORM_ID`
    - `parentFolderId` = `Your Parent Folder ID`
- **Connections:**
  - Input ← `Payment Received`
  - Output → `Validate Payment Data`
- **Failure modes / edge cases:**
  - If `parentFolderId` invalid or not shared with Drive credential → folder creation fails later.
- **Version:** typeVersion 3.4.

---

### Block B — Payment Data Validation
**Overview:** Ensures Stripe event contains critical fields before proceeding; otherwise alerts team and stops.  
**Nodes involved:** `Validate Payment Data`, `Alert Team - Payment Data Error`, `Stop - Invalid Payment Data`

#### Node: Validate Payment Data
- **Type / role:** IF node — gatekeeper validation.
- **Conditions (AND):**
  - `customer_details.email` is not empty
  - `metadata.package` is not empty
- **Connections:**
  - True → `Get row(s) in sheet`
  - False → `Alert Team - Payment Data Error`
- **Failure modes / edge cases:**
  - Expression failures if `data.object.customer_details` is undefined (depends on Stripe event/settings).
- **Version:** typeVersion 2.2.

#### Node: Alert Team - Payment Data Error
- **Type / role:** Slack node — operational alert.
- **Behavior:** Posts a message including payment id, customer email/package (or “MISSING”).
- **Channel config:** `# new-channel` (by name).
- **Connections:** Output → `Stop - Invalid Payment Data`
- **Credentials:** Slack OAuth2.
- **Failure modes:**
  - Slack channel name mismatch / bot not in channel.
  - OAuth scope missing (`chat:write`).
- **Version:** typeVersion 2.4.

#### Node: Stop - Invalid Payment Data
- **Type / role:** Stop and Error — terminates execution with a clear error.
- **Error message:** Includes payment ID and reason (missing email or package metadata).
- **Connections:** terminal.
- **Version:** typeVersion 1.

---

### Block C — CRM Lookup (Google Sheets)
**Overview:** Looks up the client record in a CRM sheet by email; stops if not found.  
**Nodes involved:** `Get row(s) in sheet`, `Check CRM Lookup Success`, `Alert Team - CRM Lookup Failed`, `Stop - CRM Lookup Failed`

#### Node: Get row(s) in sheet
- **Type / role:** Google Sheets — read/query.
- **Operation:** Get rows filtered by:
  - Lookup column: `Email`
  - Lookup value: `{{ $('Payment Received').item.json.data.object.customer_details.email }}`
- **Document:** “CRM” (Spreadsheet ID `1ROLrrdi9qCrbt80dpn27UrnXydCRH_oGKbW61_G_liw`)
- **Connections:** Output → `Check CRM Lookup Success`
- **Credentials:** Google Sheets OAuth2.
- **Failure modes / edge cases:**
  - No matching email → returns empty set; next IF fails.
  - Multiple matches: n8n will output multiple items; downstream assumes single `item` access (risk of picking first item silently).
  - Column name must match exactly `Email`.
- **Version:** typeVersion 4.7.

#### Node: Check CRM Lookup Success
- **Type / role:** IF node — ensures CRM returned required company data.
- **Condition:** `Company Legal Name` not empty from `Get row(s) in sheet`.
- **Connections:**
  - True → `Append row in sheet`
  - False → `Alert Team - CRM Lookup Failed`
- **Failure modes:** If CRM columns are renamed/missing, expression returns empty and triggers failure path.
- **Version:** typeVersion 2.2.

#### Node: Alert Team - CRM Lookup Failed
- **Type / role:** Slack alert.
- **Message:** “Client not found in CRM” with email/name/package.
- **Connections:** Output → `Stop - CRM Lookup Failed`
- **Credentials / failure modes:** Same as other Slack nodes.
- **Version:** typeVersion 2.4.

#### Node: Stop - CRM Lookup Failed
- **Type / role:** Stop and Error — halts workflow.
- **Message:** Requests manual CRM entry and retry.
- **Version:** typeVersion 1.

---

### Block D — Order Logging (Orders Sheet)
**Overview:** Appends an order record to an “Opportunities / Orders” spreadsheet, then fans out to Drive provisioning and ClickUp list creation.  
**Nodes involved:** `Append row in sheet`

#### Node: Append row in sheet
- **Type / role:** Google Sheets — write/append.
- **Operation:** Append to spreadsheet “Opportunities / Orders” (ID `1nttPA4qQHho0cRoXQy0UYmDDtBzvR3GZbCgQcDza7jw`), sheet `Sheet1` (`gid=0`).
- **Column mapping (key expressions):**
  - `price` = `Payment Received.data.object.amount_total`
  - `leadId` = `Payment Received.data.object.metadata.lead_id`
  - `paidAt` = `{{$now}}`
  - `dueDate` = `{{$json['How soon?']}}` (important: this comes from the *current item* at this point—typically the CRM row item)
  - `orderId` = `Payment Received.item.json.id`
  - `package` = `Payment Received.data.object.metadata.package`
  - `status: In Production / QA / Delivered` = `"Production"`
- **Connections:**
  - Output → `Create Client Root Folder`
  - Output → `Create a list`
- **Failure modes / edge cases:**
  - The expression `{{$json['How soon?']}}` requires the CRM sheet to have a column exactly named `How soon?`. If missing, `dueDate` will be blank.
  - Amount fields are in smallest currency unit (e.g., cents) depending on Stripe configuration; formatting may be needed.
  - Sheet schema/headers must match the mapped column IDs.
- **Version:** typeVersion 4.7.

---

### Block E — Google Drive Project Folder Provisioning
**Overview:** Creates a client root folder and standard subfolders; alerts Slack if root folder creation failed, but continues via success path only when root folder exists.  
**Nodes involved:** `Create Client Root Folder`, `Check Folder Creation Success`, `Alert Team - Folder Creation Failed`, `Create 01-Intake Folder`, `Create 02-Logo Folder`, `Create 03-Brand Kit Folder`, `Create 04-Website Folder`, `Create 05-Final Delivery Folder`, `Waits until all folders are created`

#### Node: Create Client Root Folder
- **Type / role:** Google Drive — create folder.
- **Folder name expression:**
  - `{{$now.format('yyyy-MM')}} — {{ Company Legal Name }} — {{ $json.package }}`
  - Company sourced from `Get row(s) in sheet`. Package sourced from current item (after appending order, current item includes mapped fields; `package` exists).
- **Parent folder:** `Workflow Configuration.parentFolderId`
- **Connections:**
  - Output → all subfolder creations (`Create 02...`, `Create 03...`, `Create 04...`, `Create 05...`)
  - Output → `Check Folder Creation Success`
- **Credentials:** Google Drive OAuth2.
- **Failure modes / edge cases:**
  - Invalid parent folder ID or insufficient permissions.
  - Name contains illegal characters (rare, but possible from company/package strings).
- **Version:** typeVersion 3.

#### Node: Check Folder Creation Success
- **Type / role:** IF node — checks that root folder ID exists.
- **Condition:** `Create Client Root Folder.item.json.id` not empty.
- **Connections:**
  - True → `Create 01-Intake Folder`
  - False → `Alert Team - Folder Creation Failed`
- **Failure modes:** If Drive node errors hard, execution may stop before this IF unless “Continue On Fail” is used (not configured here).
- **Version:** typeVersion 2.2.

#### Node: Alert Team - Folder Creation Failed
- **Type / role:** Slack alert.
- **Message:** Warns that root folder couldn’t be created; says workflow will continue without folder structure.
- **Connections:** none after this node (dead-end branch).
- **Important behavior note:** Despite the text claiming continuation, the configured graph does **not** route this branch back into the main path, so the “folder failed” branch ends here and does **not** proceed to email/Slack final notification.
- **Version:** typeVersion 2.4.

#### Nodes: Create 01-Intake Folder / Create 02-Logo Folder / Create 03-Brand Kit Folder / Create 04-Website Folder / Create 05-Final Delivery Folder
- **Type / role:** Google Drive — create folder (each).
- **Parent folder:** `Create Client Root Folder.item.json.id`
- **Connections:**
  - Each subfolder → `Waits until all folders are created` on a different input index.
- **Failure modes:** Same Drive permission/ID issues; individual folder creation errors will prevent the merge from completing unless configured to continue on fail (not configured).
- **Version:** typeVersion 3 for each.

#### Node: Waits until all folders are created
- **Type / role:** Merge node — synchronization barrier.
- **Mode:** `chooseBranch` with `numberInputs: 5`
- **What it does here:** Waits for all five subfolder creation branches to reach the merge, then emits one chosen branch’s data (implementation detail of `chooseBranch`).
- **Connections:** Output → `Send Welcome Email with Intake Form`
- **Failure modes / edge cases:**
  - If any subfolder branch fails/stops, merge may not emit output → email never sent.
- **Version:** typeVersion 3.2.

---

### Block F — ClickUp Project List + Task Creation
**Overview:** Creates a ClickUp list for the client project and four predefined tasks.  
**Nodes involved:** `Create a list`, `Create Task: Brand Questionnaire Review`, `Create Task: Logo Concepts`, `Create Task: Brand Kit`, `Create Task: Website Build`

#### Node: Create a list
- **Type / role:** ClickUp — create List.
- **List name expression:** `YYYY-MM — Company Legal Name — Package`
- **Workspace identifiers:**
  - Team: `9017591357`
  - Space: `90173166703`
  - Folderless: `true`
- **Connections:** Output → all four task creation nodes (fan-out).
- **Credentials:** ClickUp API (“ClickUp account 2”).
- **Failure modes / edge cases:**
  - Wrong Team/Space IDs; insufficient permissions.
  - Rate limiting if many orders arrive close together.
- **Version:** typeVersion 1.

#### Node: Create Task: Brand Questionnaire Review
- **Type / role:** ClickUp — create Task.
- **List ID:** `={{ $json.id }}` (this works because this node is directly fed by `Create a list`; `$json.id` is the list ID).
- **Fields:** status `to do`, priority 3 (High), content describes intake review.
- **Connections:** none further.
- **Failure modes:** Invalid status name if the list uses custom statuses not including `to do`.
- **Version:** typeVersion 1.

#### Node: Create Task: Logo Concepts
- **Type / role:** ClickUp — create Task.
- **List ID:** `={{ $('Create a list').item.json.id }}`
- **Fields:** status `to do`, priority 3 (High).
- **Connections:** none.
- **Version:** typeVersion 1.

#### Node: Create Task: Brand Kit
- **Type / role:** ClickUp — create Task.
- **List ID:** `Create a list.id`
- **Fields:** status `to do`, priority 2 (Normal).
- **Version:** typeVersion 1.

#### Node: Create Task: Website Build
- **Type / role:** ClickUp — create Task.
- **List ID:** `Create a list.id`
- **Fields:** status `to do`, priority 2 (Normal).
- **Version:** typeVersion 1.

---

### Block G — Client Email + Slack Team Notification + Email Failure Alert
**Overview:** Sends a welcome email containing an intake form link, validates send success, then posts a final “new kickoff” message in Slack; alerts Slack if the email fails (but does not stop the workflow).  
**Nodes involved:** `Send Welcome Email with Intake Form`, `Check Email Send Success`, `Alert Team - Email Send Failed`, `Notify Team in Slack`

#### Node: Send Welcome Email with Intake Form
- **Type / role:** Gmail — send email.
- **To:** Stripe `customer_details.email`
- **Subject:** “Welcome! Let's Get Started on Your Project 🎉”
- **HTML Body:** Uses Stripe name, package, and the configured `intakeFormUrl`.
- **Connections:** Output → `Check Email Send Success`
- **Credentials:** Gmail OAuth2.
- **Failure modes / edge cases:**
  - Gmail API scopes not granted (send permission).
  - Sending limits / throttling.
  - If Stripe email missing, validation should prevent reaching here.
- **Version:** typeVersion 2.2.

#### Node: Check Email Send Success
- **Type / role:** IF node.
- **Condition:** `Send Welcome Email with Intake Form.item.json.id` not empty (Gmail message id).
- **Connections:**
  - True → `Notify Team in Slack`
  - False → `Alert Team - Email Send Failed`
- **Failure modes:** If Gmail node errors hard, this IF may never run.
- **Version:** typeVersion 2.2.

#### Node: Alert Team - Email Send Failed
- **Type / role:** Slack alert.
- **Behavior:** Posts warning and says workflow continues (and in this case it does end here; main “notify” path is only on success).
- **Connections:** none.
- **Version:** typeVersion 2.4.

#### Node: Notify Team in Slack
- **Type / role:** Slack message — team notification of completed kickoff steps.
- **Message includes:** client name, package, amount subtotal, paid time (`$now`), and checklist.
- **Channel:** `# new-channel`
- **Credentials:** Slack OAuth2.
- **Failure modes:** same Slack channel/scope issues.
- **Version:** typeVersion 2.4.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Payment Received | Stripe Trigger | Entry point on payment success | — | Workflow Configuration | # 💳 Payment Processing (Nodes: Payment Received (Stripe Trigger), Workflow Configuration, Extract Payment Data, Get row(s) in sheet (CRM lookup), Append row in sheet (Orders) …) |
| Workflow Configuration | Set | Define constants (intake URL, parent folder ID) | Payment Received | Validate Payment Data | # 💳 Payment Processing (Nodes: Payment Received (Stripe Trigger), Workflow Configuration, Extract Payment Data, Get row(s) in sheet (CRM lookup), Append row in sheet (Orders) …) |
| Validate Payment Data | IF | Ensure Stripe payload has email + package | Workflow Configuration | Get row(s) in sheet; Alert Team - Payment Data Error | # 💳 Payment Processing (Nodes: Payment Received (Stripe Trigger), Workflow Configuration, Extract Payment Data, Get row(s) in sheet (CRM lookup), Append row in sheet (Orders) …) |
| Alert Team - Payment Data Error | Slack | Alert missing Stripe required fields | Validate Payment Data (false) | Stop - Invalid Payment Data |  |
| Stop - Invalid Payment Data | Stop and Error | Hard stop on invalid payment payload | Alert Team - Payment Data Error | — |  |
| Get row(s) in sheet | Google Sheets | CRM lookup by email | Validate Payment Data (true) | Check CRM Lookup Success | # 📊 CRM Create Client Order (Google Sheets Integration… Stripe payment → CRM lookup → Order creation → Project setup) |
| Check CRM Lookup Success | IF | Verify CRM returned company data | Get row(s) in sheet | Append row in sheet; Alert Team - CRM Lookup Failed | # 📊 CRM Create Client Order (Google Sheets Integration… Stripe payment → CRM lookup → Order creation → Project setup) |
| Alert Team - CRM Lookup Failed | Slack | Alert client not found in CRM | Check CRM Lookup Success (false) | Stop - CRM Lookup Failed |  |
| Stop - CRM Lookup Failed | Stop and Error | Hard stop if CRM lookup fails | Alert Team - CRM Lookup Failed | — |  |
| Append row in sheet | Google Sheets | Create order record | Check CRM Lookup Success (true) | Create Client Root Folder; Create a list | # 📊 CRM Create Client Order (Google Sheets Integration… Stripe payment → CRM lookup → Order creation → Project setup) |
| Create Client Root Folder | Google Drive | Create root project folder | Append row in sheet | Create 02-Logo Folder; Create 03-Brand Kit Folder; Create 04-Website Folder; Create 05-Final Delivery Folder; Check Folder Creation Success | # 📁 Google Drive Setup (Creates folder structure… Parent Folder ID configured in Workflow Configuration node…) |
| Check Folder Creation Success | IF | Verify root folder created | Create Client Root Folder | Create 01-Intake Folder; Alert Team - Folder Creation Failed | # 📁 Google Drive Setup (Creates folder structure… Parent Folder ID configured in Workflow Configuration node…) |
| Alert Team - Folder Creation Failed | Slack | Alert Drive root folder creation failure | Check Folder Creation Success (false) | — |  |
| Create 01-Intake Folder | Google Drive | Create subfolder | Check Folder Creation Success (true) | Waits until all folders are created | # 📁 Google Drive Setup (Creates folder structure… Parent Folder ID configured in Workflow Configuration node…) |
| Create 02-Logo Folder | Google Drive | Create subfolder | Create Client Root Folder | Waits until all folders are created | # 📁 Google Drive Setup (Creates folder structure… Parent Folder ID configured in Workflow Configuration node…) |
| Create 03-Brand Kit Folder | Google Drive | Create subfolder | Create Client Root Folder | Waits until all folders are created | # 📁 Google Drive Setup (Creates folder structure… Parent Folder ID configured in Workflow Configuration node…) |
| Create 04-Website Folder | Google Drive | Create subfolder | Create Client Root Folder | Waits until all folders are created | # 📁 Google Drive Setup (Creates folder structure… Parent Folder ID configured in Workflow Configuration node…) |
| Create 05-Final Delivery Folder | Google Drive | Create subfolder | Create Client Root Folder | Waits until all folders are created | # 📁 Google Drive Setup (Creates folder structure… Parent Folder ID configured in Workflow Configuration node…) |
| Waits until all folders are created | Merge | Synchronize subfolder branches | Create 01-Intake Folder; Create 02-Logo Folder; Create 03-Brand Kit Folder; Create 04-Website Folder; Create 05-Final Delivery Folder | Send Welcome Email with Intake Form | # 📁 Google Drive Setup (Creates folder structure… Parent Folder ID configured in Workflow Configuration node…) |
| Send Welcome Email with Intake Form | Gmail | Email client welcome + intake link | Waits until all folders are created | Check Email Send Success | # 📧 Client Communication (Welcome Email… Slack Notification… Note: Update intake form URL in Workflow Configuration) |
| Check Email Send Success | IF | Ensure email send returned an id | Send Welcome Email with Intake Form | Notify Team in Slack; Alert Team - Email Send Failed | # 📧 Client Communication (Welcome Email… Slack Notification… Note: Update intake form URL in Workflow Configuration) |
| Alert Team - Email Send Failed | Slack | Alert email send failure | Check Email Send Success (false) | — | # 📧 Client Communication (Welcome Email… Slack Notification… Note: Update intake form URL in Workflow Configuration) |
| Notify Team in Slack | Slack | Post final kickoff message to team | Check Email Send Success (true) | — | # 📧 Client Communication (Welcome Email… Slack Notification… Note: Update intake form URL in Workflow Configuration) |
| Create a list | ClickUp | Create ClickUp project list | Append row in sheet | Create Task: Brand Questionnaire Review; Create Task: Logo Concepts; Create Task: Brand Kit; Create Task: Website Build | # ✅ ClickUp Task Creation (Creates project list with tasks… All tasks start in "to do" status) |
| Create Task: Brand Questionnaire Review | ClickUp | Create standard task | Create a list | — | # ✅ ClickUp Task Creation (Creates project list with tasks… All tasks start in "to do" status) |
| Create Task: Logo Concepts | ClickUp | Create standard task | Create a list | — | # ✅ ClickUp Task Creation (Creates project list with tasks… All tasks start in "to do" status) |
| Create Task: Brand Kit | ClickUp | Create standard task | Create a list | — | # ✅ ClickUp Task Creation (Creates project list with tasks… All tasks start in "to do" status) |
| Create Task: Website Build | ClickUp | Create standard task | Create a list | — | # ✅ ClickUp Task Creation (Creates project list with tasks… All tasks start in "to do" status) |
| Workflow Overview | Sticky Note | Documentation | — | — | # Payment to Project Kickoff Automation (Purpose… Trigger: Stripe checkout.session.completed or invoice.payment_succeeded) |
| Payment Processing | Sticky Note | Documentation | — | — | # 💳 Payment Processing (Nodes… Key Data…) |
| Google Drive Setup | Sticky Note | Documentation | — | — | # 📁 Google Drive Setup (Creates folder structure… Folder naming format…) |
| ClickUp Task Creation | Sticky Note | Documentation | — | — | # ✅ ClickUp Task Creation (Creates project list with tasks… All tasks start in "to do" status) |
| Client Communication | Sticky Note | Documentation | — | — | # 📧 Client Communication (Welcome Email… Slack Notification… Note: Update intake form URL…) |
| CRM Lookup Note | Sticky Note | Documentation | — | — | # 📊 CRM Create Client Order (Google Sheets Integration… Data Flow…) |
| Sticky Note | Sticky Note | Documentation | — | — | # How it works (end-to-end explanation + setup steps) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n. Name it **“Payment to Project Kickoff Automation”**.

2. **Add Stripe Trigger** node named **Payment Received**  
   - Events: `checkout.session.completed`, `invoice.payment_succeeded`  
   - Connect Stripe credentials (Stripe API key).  
   - Ensure your Stripe Checkout collects `customer_details.email` and sets metadata keys like `package` (and optionally `lead_id`).

3. **Add Set** node named **Workflow Configuration**  
   - Add fields:
     - `intakeFormUrl` (string): `https://tally.so/r/YOUR_FORM_ID`
     - `parentFolderId` (string): your Google Drive parent folder ID
   - Connect: `Payment Received` → `Workflow Configuration`

4. **Add IF** node named **Validate Payment Data**  
   - Conditions (AND):
     - `{{ $('Payment Received').item.json.data.object.customer_details.email }}` **is not empty**
     - `{{ $('Payment Received').item.json.data.object.metadata.package }}` **is not empty**
   - Connect: `Workflow Configuration` → `Validate Payment Data`

5. **Add Slack** node named **Alert Team - Payment Data Error** (OAuth2)  
   - Post to channel `# new-channel` (or your channel)
   - Message uses:
     - Payment ID: `{{ $('Payment Received').item.json.id }}`
     - Email: `{{ $('Payment Received').item.json.data.object.customer_details.email || 'MISSING' }}`
     - Package: `{{ $('Payment Received').item.json.data.object.metadata.package || 'MISSING' }}`
   - Connect: `Validate Payment Data (false)` → this Slack node

6. **Add Stop and Error** node named **Stop - Invalid Payment Data**  
   - Error message referencing missing fields + payment ID
   - Connect: `Alert Team - Payment Data Error` → `Stop - Invalid Payment Data`

7. **Add Google Sheets** node named **Get row(s) in sheet** (CRM lookup)  
   - Operation: “Get row(s)” with filter:
     - Column: `Email`
     - Value: `{{ $('Payment Received').item.json.data.object.customer_details.email }}`
   - Select your CRM spreadsheet and sheet tab.
   - Connect: `Validate Payment Data (true)` → `Get row(s) in sheet`

8. **Add IF** node named **Check CRM Lookup Success**  
   - Condition: `{{ $('Get row(s) in sheet').item.json['Company Legal Name'] }}` **is not empty**
   - Connect: `Get row(s) in sheet` → `Check CRM Lookup Success`

9. **Add Slack** node named **Alert Team - CRM Lookup Failed**  
   - Post to `# new-channel`
   - Include email/name/package fields from Stripe
   - Connect: `Check CRM Lookup Success (false)` → `Alert Team - CRM Lookup Failed`

10. **Add Stop and Error** node named **Stop - CRM Lookup Failed**  
   - Connect: `Alert Team - CRM Lookup Failed` → `Stop - CRM Lookup Failed`

11. **Add Google Sheets** node named **Append row in sheet** (Orders log)  
   - Operation: Append row
   - Map columns similar to:
     - `orderId` = `{{ $('Payment Received').item.json.id }}`
     - `leadId` = `{{ $('Payment Received').item.json.data.object.metadata.lead_id }}`
     - `package` = `{{ $('Payment Received').item.json.data.object.metadata.package }}`
     - `price` = `{{ $('Payment Received').item.json.data.object.amount_total }}`
     - `paidAt` = `{{ $now }}`
     - `dueDate` = `{{ $json['How soon?'] }}` (make sure CRM row has this column)
     - `status: In Production / QA / Delivered` = `Production`
   - Connect: `Check CRM Lookup Success (true)` → `Append row in sheet`

12. **Add Google Drive** node named **Create Client Root Folder**  
   - Resource: Folder → Create
   - Parent folder ID: `{{ $('Workflow Configuration').item.json.parentFolderId }}`
   - Name: `{{ $now.format('yyyy-MM') }} — {{ $('Get row(s) in sheet').item.json['Company Legal Name'] }} — {{ $json.package }}`
   - Connect: `Append row in sheet` → `Create Client Root Folder`

13. **Add IF** node named **Check Folder Creation Success**  
   - Condition: `{{ $('Create Client Root Folder').item.json.id }}` **is not empty**
   - Connect: `Create Client Root Folder` → `Check Folder Creation Success`

14. **Add Slack** node named **Alert Team - Folder Creation Failed**  
   - Post warning to `# new-channel`
   - Connect: `Check Folder Creation Success (false)` → `Alert Team - Folder Creation Failed`

15. **Add 5 Google Drive folder creation nodes** under the root folder:
   - **Create 01-Intake Folder** (connect from `Check Folder Creation Success (true)`)
   - **Create 02-Logo Folder** (connect from `Create Client Root Folder`)
   - **Create 03-Brand Kit Folder** (connect from `Create Client Root Folder`)
   - **Create 04-Website Folder** (connect from `Create Client Root Folder`)
   - **Create 05-Final Delivery Folder** (connect from `Create Client Root Folder`)
   - Each uses parent: `{{ $('Create Client Root Folder').item.json.id }}` and the respective folder name.

16. **Add Merge** node named **Waits until all folders are created**  
   - Mode: `chooseBranch`
   - Number of inputs: 5
   - Connect each subfolder node output into a different merge input (0–4).

17. **Add Gmail** node named **Send Welcome Email with Intake Form**  
   - To: `{{ $('Payment Received').item.json.data.object.customer_details.email }}`
   - Subject and HTML message including:
     - `{{ $('Payment Received').item.json.data.object.customer_details.name }}`
     - `{{ $('Payment Received').item.json.data.object.metadata.package }}`
     - Intake link: `{{ $('Workflow Configuration').item.json.intakeFormUrl }}`
   - Connect: `Waits until all folders are created` → this Gmail node
   - Configure Gmail OAuth2 credentials.

18. **Add IF** node named **Check Email Send Success**  
   - Condition: `{{ $('Send Welcome Email with Intake Form').item.json.id }}` **is not empty**
   - Connect: Gmail node → `Check Email Send Success`

19. **Add Slack** node named **Alert Team - Email Send Failed**  
   - Connect: `Check Email Send Success (false)` → this Slack node

20. **Add Slack** node named **Notify Team in Slack**  
   - Connect: `Check Email Send Success (true)` → `Notify Team in Slack`
   - Message includes client/package/amount and checklist.
   - Configure Slack OAuth2 credentials.

21. **Add ClickUp** node named **Create a list**  
   - Operation: List → Create (folderless)
   - Name: `{{ $now.format('yyyy-MM') }} — {{ $('Get row(s) in sheet').item.json['Company Legal Name'] }} — {{ $('Payment Received').item.json.data.object.metadata.package }}`
   - Configure Team/Space IDs and ClickUp credential.
   - Connect: `Append row in sheet` → `Create a list`

22. **Add 4 ClickUp task creation nodes** connected from **Create a list**:
   - **Create Task: Brand Questionnaire Review** (list id can be `{{ $json.id }}`)
   - **Create Task: Logo Concepts** (list id `{{ $('Create a list').item.json.id }}`)
   - **Create Task: Brand Kit**
   - **Create Task: Website Build**
   - Set status `to do` and priorities (3 high for first two, 2 for last two).

23. **(Optional) Add Sticky Notes** for documentation (as in the provided workflow), and **activate**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Intake form URL must be updated in **Workflow Configuration** | `https://tally.so/r/YOUR_FORM_ID` |
| Orders sheet used: “Opportunities / Orders” | https://docs.google.com/spreadsheets/d/1nttPA4qQHho0cRoXQy0UYmDDtBzvR3GZbCgQcDza7jw/edit |
| CRM sheet used: “CRM” | https://docs.google.com/spreadsheets/d/1ROLrrdi9qCrbt80dpn27UrnXydCRH_oGKbW61_G_liw/edit |
| Operational design note: “Folder creation failed” branch does not rejoin the main path despite message claiming continuation | Consider adding a merge/continue route if you truly want to proceed without folders |
| Setup note from workflow: connect Stripe, set config node, connect CRM/ClickUp/Gmail/Slack, activate and test with Stripe test payment | Included in the “How it works” sticky note |