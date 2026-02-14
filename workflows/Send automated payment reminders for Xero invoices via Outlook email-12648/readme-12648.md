Send automated payment reminders for Xero invoices via Outlook email

https://n8nworkflows.xyz/workflows/send-automated-payment-reminders-for-xero-invoices-via-outlook-email-12648


# Send automated payment reminders for Xero invoices via Outlook email

## 1. Workflow Overview

This workflow automatically checks Xero invoices every day at noon, identifies invoices that are **not paid** and **due within the next 7 days**, sends a **payment reminder email via Microsoft Outlook** to the invoice contact, and then **logs the reminder** into the invoice’s **Xero History** for auditability.

### 1.1 Scheduled Input & Invoice Retrieval
Runs on a daily schedule and pulls invoices from Xero.

### 1.2 Eligibility Filtering (Not Paid)
Removes invoices already marked as `PAID`.

### 1.3 Due-Date Calculation & “Due Soon” Flagging
Computes `daysUntilDue` and sets a boolean `isDueSoon` when due within 7 days (and not overdue).

### 1.4 Reminder Delivery (Outlook) + Audit Logging (Xero History)
Sends the reminder email to the Xero contact email, then writes an entry in the invoice history.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Check & Fetch Invoices
**Overview:** Triggers daily and retrieves invoice data from Xero to begin processing each invoice item.  
**Nodes involved:** `Daily Invoice Check Trigger`, `Fetch All Xero Invoices`

#### Node: Daily Invoice Check Trigger
- **Type / role:** Schedule Trigger — workflow entry point.
- **Config:** Runs every day at **12:00** (local time of the n8n instance).
- **I/O connections:** Output → `Fetch All Xero Invoices`
- **Version notes:** `typeVersion 1.2` (standard schedule node behavior).
- **Edge cases / failures:**
  - Time zone expectations: noon depends on server/workspace time zone.
  - If n8n is down at trigger time, run may be missed unless n8n catch-up is configured externally.

#### Node: Fetch All Xero Invoices
- **Type / role:** Xero node — pulls invoice records.
- **Config choices:**
  - **Operation:** `getAll`
  - **Return All:** enabled (fetches all available invoices returned by the API/pagination).
- **Credentials:** Not shown in the node JSON, but Xero access is required for this node to work (OAuth2).
- **I/O connections:** Input ← `Daily Invoice Check Trigger`; Output → `Filter Out Paid Invoices`
- **Edge cases / failures:**
  - OAuth token expiration / revoked consent.
  - Large invoice volume can cause long execution times (returnAll) and potential API throttling.
  - Tenancy/organization selection issues depending on Xero credential configuration.

**Sticky note coverage (applies to this block):**  
- **Workflow Overview** (high-level description, requirements, setup, customization ideas)  
- **Step 1 Note:** “Daily Check & Filter…”

---

### Block 2 — Filter Out Paid Invoices
**Overview:** Keeps only invoices whose status is not `PAID`.  
**Nodes involved:** `Filter Out Paid Invoices`

#### Node: Filter Out Paid Invoices
- **Type / role:** IF node — conditional filter.
- **Config choices:**
  - Condition: String compare  
    - `value1 = {{ $json.Status }}`
    - `operation = notEqual`
    - `value2 = PAID`
  - “True” path means invoice is **NOT** paid and continues.
- **I/O connections:** Input ← `Fetch All Xero Invoices`; Output (true path) → `Calculate Days Until Due`
- **Edge cases / failures:**
  - If `Status` is missing or null, comparison may still pass (not equal to `PAID`) and allow unexpected items through.
  - Xero statuses can include other terminal/invalid states depending on invoice type (e.g., VOIDED); this workflow does not exclude them.

**Sticky note coverage:**  
- **Step 1 Note**

---

### Block 3 — Calculate Days Until Due & Select “Due Soon”
**Overview:** Computes how many days remain until the due date and flags invoices due in the next 7 days (0–7). Then it filters to only those flagged invoices.  
**Nodes involved:** `Calculate Days Until Due`, `Filter Invoices Due Soon`

#### Node: Calculate Days Until Due
- **Type / role:** Code node — transforms each invoice item with computed fields.
- **Config choices (JS logic):**
  - `today = new Date()`
  - `dueDate = new Date($input.item.json.DueDate)`
  - `diffDays = ceil((dueDate - today) / msPerDay)`
  - Outputs merged JSON plus:
    - `daysUntilDue: diffDays`
    - `isDueSoon: diffDays <= 7 && diffDays >= 0`
- **Key variables:** `$input.item.json.DueDate`, `daysUntilDue`, `isDueSoon`
- **I/O connections:** Input ← `Filter Out Paid Invoices`; Output → `Filter Invoices Due Soon`
- **Version notes:** `typeVersion 2` (Code node “newer” runtime style).
- **Edge cases / failures:**
  - If `DueDate` is missing or not parseable by `Date()`, `diffDays` becomes `NaN`, making `isDueSoon` false (no reminders sent).
  - Time-of-day effects: using current timestamp means invoices due “today” can produce `0` or `1` depending on time/timezone; reminders may shift by a day.
  - If you want reminders for overdue invoices, current logic explicitly excludes `diffDays < 0`.

#### Node: Filter Invoices Due Soon
- **Type / role:** IF node — conditional filter.
- **Config choices:**
  - Boolean condition:
    - `value1 = {{ $json.isDueSoon }}`
    - `value2 = true`
- **I/O connections:** Input ← `Calculate Days Until Due`; Output (true path) → `Send a message1`
- **Edge cases / failures:**
  - If the Code node didn’t set `isDueSoon`, this condition will be false and drop the item.
  - No branch is connected for “false” path (items are silently ignored).

**Sticky note coverage:**  
- **Step 2 Note**

---

### Block 4 — Send Outlook Reminder & Log to Xero History
**Overview:** Emails the invoice contact via Outlook, then writes a history record to the corresponding invoice in Xero to keep an audit trail.  
**Nodes involved:** `Send a message1`, `Log Reminder in Xero History`

#### Node: Send a message1
- **Type / role:** Microsoft Outlook node — sends an email.
- **Config choices:**
  - **Subject:** `Payment Reminder - Invoice {{ $json.InvoiceNumber }}`
  - **To:** `{{ $json.Contact.EmailAddress }}`
  - **Body:** fixed reminder text referencing `InvoiceNumber` and “due within 7 days”
- **Credentials:** Uses Microsoft Outlook OAuth2 credential named **“WFAN”**.
- **I/O connections:** Input ← `Filter Invoices Due Soon`; Output → `Log Reminder in Xero History`
- **Version notes:** `typeVersion 2`
- **Edge cases / failures:**
  - If `Contact.EmailAddress` is empty/missing, send will fail or be rejected.
  - Tenant policies (Exchange/Graph) can block sending, enforce “From” restrictions, or throttle.
  - Content is static: it does not include amount/due date despite the sticky note stating it should.

#### Node: Log Reminder in Xero History
- **Type / role:** HTTP Request node — calls Xero API endpoint not covered by the Xero node configuration here (manual call).
- **Config choices:**
  - **Method:** PUT
  - **URL:** `https://api.xero.com/api.xro/2.0/Invoices/{{ $json.InvoiceID }}/History`
  - **Authentication:** Predefined credential type → `xeroOAuth2Api` (Xero OAuth2 credential)
  - **JSON body:**
    - Writes a HistoryRecord with Details:
      - `Payment reminder email sent on {{ $now.toFormat('yyyy-MM-dd') }} - Invoice due in {{ $json.daysUntilDue }} days`
- **Key expressions/variables:** `$json.InvoiceID`, `$now.toFormat(...)`, `$json.daysUntilDue`
- **I/O connections:** Input ← `Send a message1`; no downstream nodes.
- **Version notes:** `typeVersion 4.3`
- **Edge cases / failures:**
  - Requires correct OAuth scopes and Xero tenancy context; otherwise 401/403.
  - If `InvoiceID` missing, URL becomes invalid and request fails.
  - If the Outlook node fails, this node won’t run (since it is downstream).
  - Xero API may return rate limiting or validation errors if payload format changes.

**Sticky note coverage:**  
- **Step 3 Note**

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Invoice Check Trigger | Schedule Trigger | Daily entry point at noon | — | Fetch All Xero Invoices | Step 1: Daily Check & Filter |
| Fetch All Xero Invoices | Xero | Retrieve all invoices from Xero | Daily Invoice Check Trigger | Filter Out Paid Invoices | Step 1: Daily Check & Filter |
| Filter Out Paid Invoices | IF | Exclude invoices with Status = PAID | Fetch All Xero Invoices | Calculate Days Until Due | Step 1: Daily Check & Filter |
| Calculate Days Until Due | Code | Compute daysUntilDue and isDueSoon | Filter Out Paid Invoices | Filter Invoices Due Soon | Step 2: Calculate & Identify |
| Filter Invoices Due Soon | IF | Keep invoices due within 7 days | Calculate Days Until Due | Send a message1 | Step 2: Calculate & Identify |
| Send a message1 | Microsoft Outlook | Send reminder email to contact | Filter Invoices Due Soon | Log Reminder in Xero History | Step 3: Send & Log Reminders |
| Log Reminder in Xero History | HTTP Request | Add a reminder note to Xero invoice history | Send a message1 | — | Step 3: Send & Log Reminders |
| Workflow Overview | Sticky Note | Documentation / requirements | — | — | ## Send Automated Payment Reminders for Overdue Xero Invoices… |
| Step 1 Note | Sticky Note | Documentation for block 1 | — | — | ## Step 1: Daily Check & Filter… |
| Step 2 Note | Sticky Note | Documentation for block 3 | — | — | ## Step 2: Calculate & Identify… |
| Step 3 Note | Sticky Note | Documentation for block 4 | — | — | ## Step 3: Send & Log Reminders… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: *Automated payment reminders for Xero invoices via email* (or your preferred name).
   - Keep workflow **inactive** until tested.

2. **Add Trigger: “Schedule Trigger”**
   - Node: **Schedule Trigger**
   - Set rule to run **daily at 12:00** (adjust as needed).
   - Connect → next node.

3. **Add Xero node: “Fetch All Xero Invoices”**
   - Node type: **Xero**
   - Credentials: configure **Xero OAuth2** in n8n (select organization/tenant as required).
   - Operation: **Get All**
   - Enable **Return All**
   - Connect: Schedule Trigger → Xero node.

4. **Add IF node: “Filter Out Paid Invoices”**
   - Node type: **IF**
   - Condition (String):
     - Value 1: `{{$json.Status}}`
     - Operation: **Not equal**
     - Value 2: `PAID`
   - Connect: Xero → IF (input)
   - Use the **true** output → next node (unpaid items).

5. **Add Code node: “Calculate Days Until Due”**
   - Node type: **Code**
   - Paste logic (equivalent behavior):
     - Parse `DueDate`
     - Compute `daysUntilDue`
     - Set `isDueSoon` when `0 <= daysUntilDue <= 7`
   - Connect: IF(true) → Code node.

6. **Add IF node: “Filter Invoices Due Soon”**
   - Node type: **IF**
   - Condition (Boolean):
     - Value 1: `{{$json.isDueSoon}}`
     - Equals: `true`
   - Connect: Code → IF
   - Use **true** output → email node.

7. **Add Microsoft Outlook node: “Send a message”**
   - Node type: **Microsoft Outlook**
   - Credentials: configure **Microsoft Outlook OAuth2** (Microsoft Graph) in n8n with an account allowed to send mail.
   - Set:
     - To: `{{$json.Contact.EmailAddress}}`
     - Subject: `Payment Reminder - Invoice {{$json.InvoiceNumber}}`
     - Body: your reminder text (optionally include due date/amount).
   - Connect: IF(true) → Outlook node.

8. **Add HTTP Request node: “Log Reminder in Xero History”**
   - Node type: **HTTP Request**
   - Authentication: **Predefined credential type**
     - Credential type: **Xero OAuth2 API**
   - Method: **PUT**
   - URL: `https://api.xero.com/api.xro/2.0/Invoices/{{$json.InvoiceID}}/History`
   - Body type: **JSON**
   - Body (structure):
     - `HistoryRecords` array with one object containing `Details` string
     - Include date (`$now.toFormat('yyyy-MM-dd')`) and `daysUntilDue`
   - Connect: Outlook → HTTP Request.

9. **Test the workflow**
   - Execute manually.
   - Confirm:
     - Invoices are returned.
     - Paid invoices are excluded.
     - Only invoices due within 7 days send emails.
     - Xero invoice history shows the logged entry after email send.

10. **Activate**
   - Enable the workflow to run on schedule.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Send Automated Payment Reminders for Overdue Xero Invoices… runs daily at 12 PM… sends personalized email reminders… logs reminder activity… Requirements… Setup Instructions… Customization Ideas…” | From the “Workflow Overview” sticky note (embedded workflow documentation) |
| “Step 1: Daily Check & Filter… Change the trigger time…” | Sticky note describing the schedule + paid filtering block |
| “Step 2: Calculate & Identify… Change `diffDays <= 7`…” | Sticky note describing due-soon logic customization |
| “Step 3: Send & Log Reminders… Edit the email template… add payment links…” | Sticky note describing messaging and logging block |