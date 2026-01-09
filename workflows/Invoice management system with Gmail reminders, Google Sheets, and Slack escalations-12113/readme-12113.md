Invoice management system with Gmail reminders, Google Sheets, and Slack escalations

https://n8nworkflows.xyz/workflows/invoice-management-system-with-gmail-reminders--google-sheets--and-slack-escalations-12113


# Invoice management system with Gmail reminders, Google Sheets, and Slack escalations

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow automates invoice creation and follow-up: it receives order data via a webhook, generates an invoice with calculated totals, stores it in Google Sheets, emails the invoice to the client via Gmail, then runs a daily overdue check to send escalating payment reminders (and escalates to Slack for collections at 60+ days overdue).

**Target use cases:**  
Freelancers, small businesses, and finance teams needing lightweight invoice tracking in Google Sheets with automated reminders and an escalation path for long-overdue invoices.

### 1.1 Invoice Creation (Webhook → Invoice Email)
- Accept invoice request payload (client + line items)
- Generate invoice metadata (ID, dates, terms, tax)
- Split and compute line item totals
- Aggregate line items, compute subtotal/tax/total, generate payment link
- Append invoice row to Google Sheets
- Email invoice via Gmail and return webhook response

### 1.2 Daily Overdue Check + Escalation (Schedule → Reminders/Slack → Sheet Update)
- Run daily at a configured hour
- Fetch “pending” invoices from Google Sheets
- Compute days overdue and reminder level
- Process invoices one-by-one (batch loop)
- Route to the correct reminder email (first/second/urgent/final) or Slack escalation (collections)
- Update “Last Reminder” timestamp in the sheet
- Continue loop until all overdue invoices are processed

---

## 2. Block-by-Block Analysis

### Block A — Documentation / Operator Notes (Sticky Notes)
**Overview:** Provides human guidance on intent, setup, and step grouping. Does not affect execution.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`

#### Node details
- **Sticky Note** (n8n Sticky Note)
  - **Role:** Overall workflow description, requirements, setup outline.
  - **Failure modes:** None (non-executable).
- **Sticky Note1–4** (n8n Sticky Note)
  - **Role:** Label key phases (Invoice Creation, Line Items, Daily Check, Escalation).
  - **Failure modes:** None.

---

### Block B — Invoice Intake & Initialization
**Overview:** Receives a POST request with invoice data and standardizes it into an internal invoice structure (ID, due date, tax/terms).  
**Nodes involved:** `New Invoice Request`, `Initialize Invoice`

#### 1) New Invoice Request
- **Type / role:** Webhook trigger (HTTP entry point).
- **Config (interpreted):**
  - **Path:** `POST /webhook/create-invoice` (exact base URL depends on n8n instance)
  - **Response mode:** “Respond via Respond to Webhook node” (responseNode)
- **Inputs/outputs:**
  - **Output:** `Initialize Invoice`
- **Key expectations (payload):**
  - Expects JSON body: `clientName`, `clientEmail`, `lineItems` (array), optional `paymentTerms`, optional `taxRate`.
- **Edge cases / failures:**
  - Missing/invalid JSON body or fields (expressions referencing `$json.body.*` may become `undefined`).
  - Very large payloads can hit webhook/body size limits (depends on hosting/reverse proxy).

#### 2) Initialize Invoice
- **Type / role:** Set node; normalizes and enriches incoming data.
- **Config choices:**
  - Manual mode assignments, `duplicateItem: false`.
  - Creates fields:
    - `invoiceId`: `INV-<year>-<random4digits>`
    - `clientName`, `clientEmail`, `lineItems` from webhook body
    - Defaults: `paymentTerms` to 30, `taxRate` to 0.1
    - `createdAt` = now ISO datetime
    - `dueDate` = now + paymentTerms days (ISO date)
- **Key expressions / variables:**
  - `invoiceId`: `{{ 'INV-' + $now.format('yyyy') + '-' + String(Math.floor(Math.random() * 9000) + 1000) }}`
  - `dueDate`: `{{ $now.plus({ days: $json.body.paymentTerms || 30 }).toISODate() }}`
- **Inputs/outputs:**
  - Input: `New Invoice Request`
  - Output: `Split Line Items`
- **Edge cases / failures:**
  - `lineItems` not an array → downstream Split Out may fail or produce unexpected output.
  - Random invoice ID is not collision-proof (possible duplicates over time).

---

### Block C — Line Item Processing & Invoice Calculation
**Overview:** Breaks line items into individual items, computes per-line totals, aggregates them back, and builds final invoice totals and payment link.  
**Nodes involved:** `Split Line Items`, `Calculate Line Total`, `Aggregate Line Items`, `Build Invoice Object`

#### 1) Split Line Items
- **Type / role:** Split Out node; iterates over `lineItems` array.
- **Config choices:**
  - Field to split: `lineItems`
  - Include all other fields (so invoice metadata stays with each split item)
- **Inputs/outputs:**
  - Input: `Initialize Invoice`
  - Output: `Calculate Line Total`
- **Edge cases / failures:**
  - If `lineItems` is missing/null/not an array, the node may output zero items or error depending on n8n behavior/version.

#### 2) Calculate Line Total
- **Type / role:** Code node; computes quantity × rate for each line item and reshapes fields.
- **Logic summary:**
  - Reads one item (`$input.first().json`) and expects `lineItems` to be the current split element.
  - Defaults: quantity=1, rate=0.
  - Outputs normalized structure with `lineItem` object including `lineTotal`.
- **Inputs/outputs:**
  - Input: `Split Line Items` (one item per line)
  - Output: `Aggregate Line Items`
- **Failure modes / edge cases:**
  - If `lineItems` lacks `quantity/rate/description`, defaults apply (could produce $0 lines).
  - Non-numeric rate/quantity → JS multiplication yields `NaN`, which then propagates to totals.

#### 3) Aggregate Line Items
- **Type / role:** Aggregate node; combines all items back into a single item.
- **Config choices:**
  - Aggregation mode: “aggregate all item data”
  - Destination field: `processedLines`
- **Inputs/outputs:**
  - Input: `Calculate Line Total` (many items)
  - Output: `Build Invoice Object` (single aggregated item)
- **Edge cases / failures:**
  - If no line items were produced, `processedLines` may be empty; downstream code must handle it.

#### 4) Build Invoice Object
- **Type / role:** Code node; computes subtotal/tax/total and builds final invoice payload.
- **Logic summary:**
  - Reads aggregated `processedLines`.
  - Extracts shared fields from the first line (invoiceId, client, etc.).
  - `subtotal` = sum of `lineItem.lineTotal`
  - `tax` = rounded to 2 decimals
  - `total` = subtotal + tax
  - `paymentLink` = `https://pay.company.com/invoice/<invoiceId>`
  - Sets `status: 'pending'`, `createdAt: new Date().toISOString()`
- **Inputs/outputs:**
  - Input: `Aggregate Line Items`
  - Output: `Save Invoice to Sheet`
- **Edge cases / failures:**
  - If `processedLines` is empty, invoiceId becomes `'unknown'` and totals become 0; invoice still gets created unless additional validation is added.
  - Payment link domain is hard-coded; must be changed for real deployments.

---

### Block D — Persistence, Invoice Email, and Webhook Response
**Overview:** Stores the invoice in Google Sheets, emails the invoice via Gmail, and responds to the original webhook request with a JSON result.  
**Nodes involved:** `Save Invoice to Sheet`, `Send Invoice Email`, `Invoice Created Response`

#### 1) Save Invoice to Sheet
- **Type / role:** Google Sheets node; appends a new invoice row.
- **Config choices:**
  - Operation: **Append**
  - Sheet name: `Invoices`
  - Document ID: not set in JSON (must be selected in n8n)
  - Column mapping (defines values):
    - `Invoice ID`, `Client`, `Email`, `Subtotal`, `Tax`, `Total`, `Created`, `Due Date`, `Status`
    - `Last Reminder` set to empty string
- **Inputs/outputs:**
  - Input: `Build Invoice Object`
  - Output: `Send Invoice Email`
- **Version-specific notes:**
  - Uses Google Sheets node v4.5 style mapping.
- **Failure modes / edge cases:**
  - Missing credentials / insufficient permissions to the sheet.
  - Sheet/tab `Invoices` or expected columns not present → append may fail or write to wrong columns.
  - Document ID unset will block execution until configured.

#### 2) Send Invoice Email
- **Type / role:** Gmail node; sends invoice email to client.
- **Config choices:**
  - To: `clientEmail`
  - Subject/body templated with invoice ID, total, due date, payment link.
- **Inputs/outputs:**
  - Input: `Save Invoice to Sheet`
  - Output: `Invoice Created Response`
- **Failure modes / edge cases:**
  - Gmail OAuth not configured, token expired, or sending limits reached.
  - `total.toFixed(2)` will fail if `total` is not a number (ensure `total` is numeric in Build step).

#### 3) Invoice Created Response
- **Type / role:** Respond to Webhook; returns response to caller.
- **Config choices:**
  - Respond with JSON string: `{ success: true, invoiceId, total, dueDate }`
- **Inputs/outputs:**
  - Input: `Send Invoice Email`
  - Output: ends execution for webhook branch
- **Edge cases / failures:**
  - If earlier nodes fail and workflow stops, webhook call will time out unless error workflow/handling is configured.
  - Response body uses `JSON.stringify(...)` but “Respond with: json” is selected; you may prefer returning an object directly to avoid double-encoding (depends on n8n node behavior/version).

---

### Block E — Daily Overdue Detection
**Overview:** Runs daily, retrieves pending invoices from Sheets, computes overdue days and assigns an escalation level.  
**Nodes involved:** `Daily Overdue Check`, `Get Unpaid Invoices`, `Calculate Overdue Days`

#### 1) Daily Overdue Check
- **Type / role:** Schedule Trigger; time-based entry point.
- **Config choices:**
  - Runs daily at **09:00** (based on “triggerAtHour: 9”).
- **Inputs/outputs:**
  - Output: `Get Unpaid Invoices`
- **Edge cases / failures:**
  - Time zone depends on n8n instance settings; ensure intended timezone is configured.

#### 2) Get Unpaid Invoices
- **Type / role:** Google Sheets node; fetches rows with `Status = pending`.
- **Config choices:**
  - Operation: **Get Many**
  - Filter: `Status` equals `pending`
  - Sheet name: `Invoices`
  - Document ID: not set in JSON (must be selected)
- **Inputs/outputs:**
  - Input: `Daily Overdue Check`
  - Output: `Calculate Overdue Days`
- **Failure modes / edge cases:**
  - Same Google Sheets credential and schema risks as the append node.
  - If there are many invoices, can be slow or hit API limits.

#### 3) Calculate Overdue Days
- **Type / role:** Code node; computes days overdue and maps to reminder levels.
- **Logic summary:**
  - For each sheet row:
    - `daysOverdue = floor((today - dueDate)/1day)`
    - Reminder levels:
      - 1–7 days: `first`
      - 8–14: `second`
      - 15–30: `urgent`
      - 31–59: `final`
      - 60+: `collections`
      - Otherwise: `none`
  - Filters out non-overdue invoices (`daysOverdue > 0`)
  - Outputs one item per overdue invoice with normalized fields.
- **Inputs/outputs:**
  - Input: `Get Unpaid Invoices`
  - Output: `Loop Over Invoices`
- **Edge cases / failures:**
  - Invalid/missing `Due Date` in sheet → `new Date(...)` can produce `Invalid Date`, making `daysOverdue` become `NaN`; such items will not pass `> 0` but could hide data issues.
  - Uses local server time; consider consistent timezones and due date formatting.

---

### Block F — Reminder Loop, Escalation Routing, Notifications, and Sheet Update
**Overview:** Processes overdue invoices one at a time, routes to the correct reminder action, then updates the sheet with the last reminder timestamp and loops until completion.  
**Nodes involved:** `Loop Over Invoices`, `Route by Reminder Level`, `First Reminder Email`, `Second Reminder Email`, `Urgent Reminder Email`, `Final Notice Email`, `Escalate to Collections`, `Merge Reminder Paths`, `Update Reminder Date`, `Continue Loop`

#### 1) Loop Over Invoices
- **Type / role:** Split In Batches; iterates invoices sequentially.
- **Config choices:**
  - Batch size: 1
- **Connections:**
  - Main output 0 → `Route by Reminder Level`
  - Main output 1 is unused (commonly indicates “no items left” path)
  - Loop-back is implemented via `Continue Loop` → `Loop Over Invoices`
- **Edge cases / failures:**
  - If any reminder branch errors, the loop stops unless error handling is configured.
  - Large invoice sets can take long; consider rate limits and execution timeouts.

#### 2) Route by Reminder Level
- **Type / role:** Switch; branches based on `reminderLevel`.
- **Config choices:**
  - Rules for outputs: First/Second/Urgent/Final/Collections
  - Fallback output: `none` (configured, but not connected to any node)
- **Inputs/outputs:**
  - Input: `Loop Over Invoices`
  - Outputs:
    - First → `First Reminder Email`
    - Second → `Second Reminder Email`
    - Urgent → `Urgent Reminder Email`
    - Final → `Final Notice Email`
    - Collections → `Escalate to Collections`
- **Edge cases / failures:**
  - If reminderLevel is `none` (or unexpected), item goes to fallback output which is not connected → processing halts for that item (and may stall the loop logic depending on execution semantics). Consider connecting fallback to `Merge Reminder Paths` or directly to `Update Reminder Date`/`Continue Loop`.

#### 3) First Reminder Email / Second Reminder Email / Urgent Reminder Email / Final Notice Email
- **Type / role:** Gmail nodes; send progressively stronger reminders.
- **Config choices:**
  - To: `clientEmail`
  - Message includes invoice ID, amount, due date, and (for later stages) days overdue.
- **Inputs/outputs:**
  - Each outputs into `Merge Reminder Paths`
- **Edge cases / failures:**
  - Gmail rate limits; bounced addresses; missing `clientEmail`.
  - `total.toFixed(2)` requires numeric `total` (this is ensured by parseFloat in overdue calc, but can be `NaN` if sheet contains non-numeric total).

#### 4) Escalate to Collections
- **Type / role:** Slack node; posts a message to a channel for 60+ days overdue.
- **Config choices:**
  - Resource: message; operation: post
  - Channel: `#collections` (selected by name)
  - Message includes invoice ID/client/amount/days overdue
- **Inputs/outputs:**
  - Input: Switch “Collections”
  - Output: `Merge Reminder Paths`
- **Failure modes / edge cases:**
  - Slack credential issues, missing scopes (chat:write), channel not found, workspace restrictions.

#### 5) Merge Reminder Paths
- **Type / role:** Merge node; reunifies all reminder/escalation branches.
- **Config choices:**
  - Mode: “chooseBranch” (passes through whichever input arrives)
- **Inputs/outputs:**
  - Inputs: any of the reminder/slack nodes
  - Output: `Update Reminder Date`
- **Edge cases / failures:**
  - If a branch never reaches the merge (e.g., Switch fallback output), the flow for that invoice won’t continue.

#### 6) Update Reminder Date
- **Type / role:** Google Sheets node; updates the invoice row with last reminder timestamp.
- **Config choices:**
  - Operation: **Update**
  - Sheet: `Invoices`
  - Updates columns:
    - `Invoice ID` = current invoiceId
    - `Last Reminder` = now ISO timestamp
- **Important behavior note:**
  - Google Sheets “Update” typically needs a row identifier or a configured matching column. In many n8n setups, Update requires the internal row number or a “key column” mapping. As configured here, it *assumes* the node can locate the row based on provided fields—verify in your n8n Google Sheets node UI that “Update” is configured to match by `Invoice ID`.
- **Inputs/outputs:**
  - Input: `Merge Reminder Paths`
  - Output: `Continue Loop`
- **Failure modes / edge cases:**
  - If row matching is not configured, update may fail or update the wrong row.
  - Document ID unset will block execution.

#### 7) Continue Loop
- **Type / role:** Code node; passthrough used to re-trigger the next batch.
- **Config:**
  - `return $input.all();`
- **Inputs/outputs:**
  - Input: `Update Reminder Date`
  - Output: loops back to `Loop Over Invoices`
- **Edge cases:**
  - If input is empty, loop behavior depends on SplitInBatches semantics; typically it will proceed to “no items left” output (which is currently unused).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Operator overview, requirements, setup notes | — | — | ## Generate invoices and send payment reminders with escalation / Who is this for? / What this workflow does / Setup / Requirements |
| Sticky Note1 | Sticky Note | Labels Step 1 block | — | — | **Step 1: Invoice Creation** Receive order data, generate invoice ID, and initialize invoice fields. |
| Sticky Note2 | Sticky Note | Labels Step 2 block | — | — | **Step 2: Line Item Processing** Split Out each line item, calculate totals, and Aggregate results. |
| Sticky Note3 | Sticky Note | Labels Step 3 block | — | — | **Step 3: Daily Overdue Check** Schedule Trigger runs daily. Loop Over batches each invoice for processing. |
| Sticky Note4 | Sticky Note | Labels Step 4 block | — | — | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| New Invoice Request | Webhook | Receives invoice creation requests | — | Initialize Invoice | **Step 1: Invoice Creation** Receive order data, generate invoice ID, and initialize invoice fields. |
| Initialize Invoice | Set | Generates invoice ID, sets defaults, computes due date | New Invoice Request | Split Line Items | **Step 1: Invoice Creation** Receive order data, generate invoice ID, and initialize invoice fields. |
| Split Line Items | Split Out | Iterates over line items array | Initialize Invoice | Calculate Line Total | **Step 2: Line Item Processing** Split Out each line item, calculate totals, and Aggregate results. |
| Calculate Line Total | Code | Computes per-line totals and normalizes structure | Split Line Items | Aggregate Line Items | **Step 2: Line Item Processing** Split Out each line item, calculate totals, and Aggregate results. |
| Aggregate Line Items | Aggregate | Aggregates processed lines into array | Calculate Line Total | Build Invoice Object | **Step 2: Line Item Processing** Split Out each line item, calculate totals, and Aggregate results. |
| Build Invoice Object | Code | Computes subtotal/tax/total and payment link | Aggregate Line Items | Save Invoice to Sheet | **Step 2: Line Item Processing** Split Out each line item, calculate totals, and Aggregate results. |
| Save Invoice to Sheet | Google Sheets | Appends invoice to sheet | Build Invoice Object | Send Invoice Email |  |
| Send Invoice Email | Gmail | Emails invoice to client | Save Invoice to Sheet | Invoice Created Response |  |
| Invoice Created Response | Respond to Webhook | Returns success payload to caller | Send Invoice Email | — |  |
| Daily Overdue Check | Schedule Trigger | Runs daily to start reminder processing | — | Get Unpaid Invoices | **Step 3: Daily Overdue Check** Schedule Trigger runs daily. Loop Over batches each invoice for processing. |
| Get Unpaid Invoices | Google Sheets | Fetches pending invoices | Daily Overdue Check | Calculate Overdue Days | **Step 3: Daily Overdue Check** Schedule Trigger runs daily. Loop Over batches each invoice for processing. |
| Calculate Overdue Days | Code | Computes days overdue and reminder level | Get Unpaid Invoices | Loop Over Invoices | **Step 3: Daily Overdue Check** Schedule Trigger runs daily. Loop Over batches each invoice for processing. |
| Loop Over Invoices | Split In Batches | Processes overdue invoices sequentially | Calculate Overdue Days; Continue Loop | Route by Reminder Level | **Step 3: Daily Overdue Check** Schedule Trigger runs daily. Loop Over batches each invoice for processing. |
| Route by Reminder Level | Switch | Routes to reminder/escalation actions | Loop Over Invoices | First Reminder Email; Second Reminder Email; Urgent Reminder Email; Final Notice Email; Escalate to Collections | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| First Reminder Email | Gmail | Sends first reminder | Route by Reminder Level | Merge Reminder Paths | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| Second Reminder Email | Gmail | Sends second reminder | Route by Reminder Level | Merge Reminder Paths | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| Urgent Reminder Email | Gmail | Sends urgent reminder | Route by Reminder Level | Merge Reminder Paths | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| Final Notice Email | Gmail | Sends final notice | Route by Reminder Level | Merge Reminder Paths | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| Escalate to Collections | Slack | Posts escalation to Slack channel | Route by Reminder Level | Merge Reminder Paths | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| Merge Reminder Paths | Merge | Re-joins branches after reminder action | First/Second/Urgent/Final/Collections nodes | Update Reminder Date | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| Update Reminder Date | Google Sheets | Updates “Last Reminder” timestamp | Merge Reminder Paths | Continue Loop | **Step 4: Reminder Escalation** Switch routes to 5 reminder levels based on days overdue. Collections at 60+ days. |
| Continue Loop | Code | Pass-through to continue SplitInBatches loop | Update Reminder Date | Loop Over Invoices | **Step 3: Daily Overdue Check** Schedule Trigger runs daily. Loop Over batches each invoice for processing. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it similarly to: “Generate invoices and send payment reminders with escalation using Google Sheets”.

2) **Add documentation sticky notes (optional but recommended)**
   - Add 5 Sticky Note nodes and paste the provided contents:
     - Overall overview note + Step 1–4 notes.

### A. Invoice creation branch

3) **Add Webhook node**
   - Node type: **Webhook**
   - Name: `New Invoice Request`
   - Method: **POST**
   - Path: `create-invoice`
   - Response mode: **Respond to Webhook node**

4) **Add Set node**
   - Type: **Set**
   - Name: `Initialize Invoice`
   - Add fields (expressions):
     - `invoiceId` (string): `{{ 'INV-' + $now.format('yyyy') + '-' + String(Math.floor(Math.random() * 9000) + 1000) }}`
     - `clientName` (string): `{{ $json.body.clientName }}`
     - `clientEmail` (string): `{{ $json.body.clientEmail }}`
     - `lineItems` (object/array): `{{ $json.body.lineItems }}`
     - `paymentTerms` (number): `{{ $json.body.paymentTerms || 30 }}`
     - `taxRate` (number): `{{ $json.body.taxRate || 0.1 }}`
     - `createdAt` (string): `{{ $now.toISO() }}`
     - `dueDate` (string): `{{ $now.plus({ days: $json.body.paymentTerms || 30 }).toISODate() }}`

5) **Add Split Out node**
   - Type: **Split Out**
   - Name: `Split Line Items`
   - Field to split out: `lineItems`
   - Enable: “Include all other fields”

6) **Add Code node (line totals)**
   - Type: **Code** (JavaScript)
   - Name: `Calculate Line Total`
   - Paste logic that:
     - Reads `lineItems.quantity`, `lineItems.rate`, computes `lineTotal`
     - Outputs `{ invoiceId, clientName, clientEmail, taxRate, dueDate, lineItem: {...} }`

7) **Add Aggregate node**
   - Type: **Aggregate**
   - Name: `Aggregate Line Items`
   - Mode: aggregate all item data
   - Destination field: `processedLines`

8) **Add Code node (invoice object)**
   - Type: **Code**
   - Name: `Build Invoice Object`
   - Compute `subtotal`, `tax`, `total`, `paymentLink`
   - Output invoice fields:
     - `invoiceId, clientName, clientEmail, lineItems, subtotal, taxRate, tax, total, dueDate, paymentLink, status='pending', createdAt`

9) **Add Google Sheets node (append)**
   - Type: **Google Sheets**
   - Name: `Save Invoice to Sheet`
   - Credentials: connect a Google account with access to the target spreadsheet
   - Operation: **Append**
   - Document: select your spreadsheet (set **Document ID**)
   - Sheet/tab: `Invoices`
   - Ensure the sheet has columns matching:
     - `Invoice ID`, `Client`, `Email`, `Subtotal`, `Tax`, `Total`, `Created`, `Due Date`, `Status`, `Last Reminder`
   - Map fields from the invoice object; set `Last Reminder` to empty.

10) **Add Gmail node (send invoice)**
   - Type: **Gmail**
   - Name: `Send Invoice Email`
   - Credentials: Gmail OAuth2
   - To: `{{ $json.clientEmail }}`
   - Subject/body use invoice fields and payment link (as in the workflow).

11) **Add Respond to Webhook node**
   - Type: **Respond to Webhook**
   - Name: `Invoice Created Response`
   - Respond with: JSON
   - Response body: include `success`, `invoiceId`, `total`, `dueDate`

12) **Connect nodes (invoice branch)**
   - `New Invoice Request` → `Initialize Invoice` → `Split Line Items` → `Calculate Line Total` → `Aggregate Line Items` → `Build Invoice Object` → `Save Invoice to Sheet` → `Send Invoice Email` → `Invoice Created Response`

### B. Daily reminders branch

13) **Add Schedule Trigger**
   - Type: **Schedule Trigger**
   - Name: `Daily Overdue Check`
   - Configure to run daily at **09:00** (adjust timezone as needed)

14) **Add Google Sheets node (get many)**
   - Type: **Google Sheets**
   - Name: `Get Unpaid Invoices`
   - Operation: **Get Many**
   - Document: same spreadsheet (Document ID required)
   - Sheet: `Invoices`
   - Filter: `Status` equals `pending`

15) **Add Code node (overdue calculation)**
   - Type: **Code**
   - Name: `Calculate Overdue Days`
   - For each row:
     - Parse `Due Date`
     - Compute `daysOverdue`
     - Assign `reminderLevel` using thresholds (1/8/15/31/60)
     - Output only items where `daysOverdue > 0`
   - Normalize output fields: `invoiceId, clientName, clientEmail, total, dueDate, daysOverdue, reminderLevel`

16) **Add Split In Batches**
   - Type: **Split In Batches**
   - Name: `Loop Over Invoices`
   - Batch size: **1**

17) **Add Switch node**
   - Type: **Switch**
   - Name: `Route by Reminder Level`
   - Add rules:
     - If `{{ $json.reminderLevel }}` equals `first` → output “First”
     - equals `second` → “Second”
     - equals `urgent` → “Urgent”
     - equals `final` → “Final”
     - equals `collections` → “Collections”
   - Fallback: `none` (consider connecting it to a safe path)

18) **Add Gmail reminder nodes (4x)**
   - Type: **Gmail**
   - Names:
     - `First Reminder Email`
     - `Second Reminder Email`
     - `Urgent Reminder Email`
     - `Final Notice Email`
   - Each sends to `{{ $json.clientEmail }}` with increasing urgency, referencing `invoiceId`, `total`, `dueDate`, `daysOverdue`.

19) **Add Slack node (collections escalation)**
   - Type: **Slack**
   - Name: `Escalate to Collections`
   - Credentials: Slack OAuth with permission to post messages
   - Operation: Post message to channel `#collections`
   - Message includes invoice metadata and days overdue

20) **Add Merge node**
   - Type: **Merge**
   - Name: `Merge Reminder Paths`
   - Mode: **Choose Branch**

21) **Add Google Sheets node (update last reminder)**
   - Type: **Google Sheets**
   - Name: `Update Reminder Date`
   - Operation: **Update**
   - Document/sheet: same as above
   - Set `Last Reminder` to `{{ $now.toISO() }}`
   - Ensure update is configured to match the correct row (commonly by a key column such as `Invoice ID`).

22) **Add Code node (continue loop)**
   - Type: **Code**
   - Name: `Continue Loop`
   - Code: `return $input.all();`

23) **Connect nodes (reminder branch)**
   - `Daily Overdue Check` → `Get Unpaid Invoices` → `Calculate Overdue Days` → `Loop Over Invoices` → `Route by Reminder Level`
   - Switch outputs:
     - First → `First Reminder Email` → `Merge Reminder Paths`
     - Second → `Second Reminder Email` → `Merge Reminder Paths`
     - Urgent → `Urgent Reminder Email` → `Merge Reminder Paths`
     - Final → `Final Notice Email` → `Merge Reminder Paths`
     - Collections → `Escalate to Collections` → `Merge Reminder Paths`
   - `Merge Reminder Paths` → `Update Reminder Date` → `Continue Loop` → (back to) `Loop Over Invoices`

24) **Configure credentials**
   - **Google Sheets credential:** access to the invoice spreadsheet.
   - **Gmail credential:** permission to send emails.
   - **Slack credential:** permission to post to the target channel.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create Google Sheet with invoice columns | From overview sticky note: “Create Google Sheet with invoice columns” |
| Connect Google Sheets and Gmail | From overview sticky note: required to append invoices + send emails |
| Configure Slack for escalations | From overview sticky note: collections notifications |
| Requirements: Google Sheets, Gmail account, Slack workspace | From overview sticky note |

