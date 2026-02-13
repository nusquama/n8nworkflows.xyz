Deliver encrypted payroll PDFs via Google Drive, Sheets, Gmail and Slack

https://n8nworkflows.xyz/workflows/deliver-encrypted-payroll-pdfs-via-google-drive--sheets--gmail-and-slack-12660


# Deliver encrypted payroll PDFs via Google Drive, Sheets, Gmail and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow automates secure payroll distribution. When a new payslip PDF arrives in Google Drive, it downloads the file, looks up the employee’s security metadata (email + National ID) in Google Sheets, verifies the metadata exists, encrypts the PDF with AES‑128, and then emails the encrypted PDF to the employee via Gmail. If encryption fails (or if required metadata is missing), it alerts administrators via Slack.

**Typical use cases**
- Monthly payroll delivery where PDFs must be encrypted per employee.
- Compliance-oriented distribution requiring auditable failure alerts.
- Minimizing manual handling of unencrypted payroll artifacts.

### 1.1 Logical Blocks
1. **Input Reception (Drive watch + download)**
2. **Employee Metadata Lookup (Sheets lookup)**
3. **Security Validation (integrity check / fail fast)**
4. **Encryption & Success Routing (encrypt + IF)**
5. **Delivery & Incident Alerting (Gmail send vs Slack alert)**
6. **Documentation / Notes (Sticky Notes)**

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Drive watch + download)
**Overview:** Detects new files in a Google Drive location and downloads the PDF as binary for downstream processing.

**Nodes Involved**
- Trigger: Watch Secure Inbox
- Download PDF Binary

#### Node: Trigger: Watch Secure Inbox
- **Type / role:** Google Drive node (acts as the workflow trigger by watching a Drive location).
- **Configuration (interpreted):**
  - Drive: **My Drive**
  - Folder: **root** (Root folder)
  - Operation: effectively “watch for new file” (the node name indicates watch semantics; configuration specifies drive/folder scope).
- **Key variables/fields used downstream:** outputs a file object including `id` (used later to download).
- **Connections:**
  - **Output →** Download PDF Binary
- **Version notes:** `typeVersion: 3` for Google Drive node.
- **Potential failures / edge cases:**
  - OAuth permission issues (Drive scope, revoked token).
  - Watching root can be noisy (many unrelated file events).
  - If the watch returns non-PDF files, the downstream “download + encrypt” may fail (no explicit MIME/type filtering is present).

#### Node: Download PDF Binary
- **Type / role:** Google Drive node (download operation; retrieves the file content as binary).
- **Configuration (interpreted):**
  - Operation: **download**
  - File ID: `={{ $json.id }}` (download the file emitted by the trigger)
- **Connections:**
  - **Input ←** Trigger: Watch Secure Inbox
  - **Output →** Lookup Employee Password
- **Version notes:** `typeVersion: 3`.
- **Potential failures / edge cases:**
  - File not accessible (permissions) or removed between trigger and download.
  - Large PDFs can hit timeouts depending on n8n and Drive limits.
  - Output binary property naming must match what the encryption node expects; this workflow does not explicitly map binary field names.

---

### Block 2 — Employee Metadata Lookup (Sheets lookup)
**Overview:** Finds employee-specific security metadata (email + National ID) from a Google Sheet to drive encryption and delivery.

**Nodes Involved**
- Lookup Employee Password

#### Node: Lookup Employee Password
- **Type / role:** Google Sheets node (lookup row).
- **Configuration (interpreted):**
  - Operation: **lookup**
  - Document ID: `YOUR_EMPLOYEE_DB_ID` (placeholder; must be replaced)
  - Expected columns (implied by later usage): `Email`, `National_ID`
- **Connections:**
  - **Input ←** Download PDF Binary
  - **Output →** Pre-Encryption Integrity Check
- **Version notes:** `typeVersion: 4`.
- **Key expressions/variables used later:**
  - `$node["Lookup Employee Password"].json.Email`
  - `$input.first().json.National_ID` (checked in the next node)
- **Potential failures / edge cases:**
  - Missing/incorrect document ID or permissions.
  - Lookup key is not shown in JSON; if not configured properly, it may return no rows.
  - Column name mismatches (`National_ID` vs `National ID`, casing, spaces).
  - Multiple matches could return multiple items; the workflow later assumes `.first()` semantics.

---

### Block 3 — Security Validation (integrity check / fail fast)
**Overview:** Ensures required security metadata exists before encryption; otherwise throws an error to prevent insecure delivery.

**Nodes Involved**
- Pre-Encryption Integrity Check

#### Node: Pre-Encryption Integrity Check
- **Type / role:** Code node (JavaScript validation gate).
- **Configuration (interpreted):**
  - JS logic checks whether `National_ID` exists on the first input item.
  - Throws an error if missing: `"Employee Security Metadata Missing"`.
  - Returns the first item otherwise.
- **Code (key logic):**
  - `if (!$input.first().json.National_ID) { throw new Error("Employee Security Metadata Missing"); }`
  - `return $input.first();`
- **Connections:**
  - **Input ←** Lookup Employee Password
  - **Output →** Encrypt PDF (AES-128)
- **Version notes:** `typeVersion: 2`.
- **Potential failures / edge cases:**
  - If Sheets returns zero items, `$input.first()` can be `undefined` and itself throw (different error than intended).
  - If multiple items come in (multiple matches), this node discards all but the first.
  - This error is not explicitly routed to the Slack node (Slack is only reached via the IF node); depending on n8n error handling settings, the workflow run may fail without sending Slack alert.

---

### Block 4 — Encryption & Success Routing (encrypt + IF)
**Overview:** Encrypts the PDF and routes execution based on whether encryption produced expected output.

**Nodes Involved**
- Encrypt PDF (AES-128)
- IF: Encryption Success?

#### Node: Encrypt PDF (AES-128)
- **Type / role:** HTMLCSS to PDF node (PDF manipulation: encryption).
- **Configuration (interpreted):**
  - Resource: **pdfManipulation**
  - Operation: **encrypt**
  - Error handling: `onError = continueErrorOutput` (the workflow continues and provides an error output object instead of hard-failing the run).
- **Important missing mapping (implementation note):**
  - The sticky note states the password is “dynamically mapped to the National ID column”, but the node parameters shown do not expose that mapping. In practice, this node typically needs a password field (often via expression) pointing to `{{$node["Lookup Employee Password"].json.National_ID}}` (or similar) and must be configured to use the incoming binary PDF.
- **Connections:**
  - **Input ←** Pre-Encryption Integrity Check
  - **Output →** IF: Encryption Success?
- **Version notes:** `typeVersion: 1`.
- **Potential failures / edge cases:**
  - Missing/incorrect API credentials for the HTMLCSS to PDF service.
  - Missing binary data input (if the node expects a specific binary property name).
  - Password not supplied or invalid password constraints.
  - Service limits/timeouts for encryption.
  - Because `continueErrorOutput` is enabled, failures may surface as missing `data` rather than a thrown error.

#### Node: IF: Encryption Success?
- **Type / role:** IF node (branching).
- **Configuration (interpreted):**
  - Condition: boolean check `={{ !!$json.data }}`  
    Meaning: treat encryption as successful if the current item has a truthy `data` field.
- **Connections:**
  - **True (output 0) →** Deliver Secure Email
  - **False (output 1) →** Alert Admin (Slack)
- **Version notes:** `typeVersion: 1`.
- **Potential failures / edge cases:**
  - The success predicate depends on encryption node’s response schema (`data`). If the encryption node outputs binary instead (or uses another key), this check may incorrectly route to Slack.
  - If encryption fails and still returns a `data` field with an error payload, it may incorrectly route to Gmail.

---

### Block 5 — Delivery & Incident Alerting (Gmail send vs Slack alert)
**Overview:** On success, emails the encrypted payslip to the employee; on failure, alerts admins in Slack with context.

**Nodes Involved**
- Deliver Secure Email
- Alert Admin (Slack)

#### Node: Deliver Secure Email
- **Type / role:** Gmail node (send email).
- **Configuration (interpreted):**
  - To: `={{ $node["Lookup Employee Password"].json.Email }}`
  - Subject: `Secure Payslip: {{ $now.minus({ months: 1 }).toFormat('MMMM yyyy') }}`
  - Message body: instructs employee to use National ID to unlock.
  - Attachments: **not explicitly configured in shown parameters**. For real delivery, the encrypted PDF binary must be attached (usually via “Attachments” options mapping to the correct binary property).
- **Connections:**
  - **Input ←** IF: Encryption Success? (true branch)
  - **Output →** none
- **Version notes:** `typeVersion: 2.1`.
- **Potential failures / edge cases:**
  - Gmail OAuth token revoked / insufficient scopes.
  - Missing attachment mapping: email may send without the PDF.
  - Invalid email address from Sheets lookup.
  - Rate limits / daily send limits.

#### Node: Alert Admin (Slack)
- **Type / role:** Slack node (send message to user or channel depending on configuration).
- **Configuration (interpreted):**
  - Text:
    - Includes employee email from Sheets node
    - Generic reason: “Encryption failed or metadata missing.”
  - Authentication: OAuth2
  - “select: user” but `user.value` is empty in JSON; depending on Slack node behavior, this may fail or require a channel instead.
- **Connections:**
  - **Input ←** IF: Encryption Success? (false branch)
  - **Output →** none
- **Version notes:** `typeVersion: 2.1`.
- **Potential failures / edge cases:**
  - Slack OAuth scopes missing (chat:write, etc.).
  - Destination not specified (empty user selection).
  - Message references Lookup node; if failure occurred before lookup (or lookup failed), the expression may resolve to empty and/or error.

---

### Block 6 — Documentation / Notes (Sticky Notes)
**Overview:** Provides embedded operator guidance and security intent.

**Nodes Involved**
- Main Documentation
- Security Overview

#### Node: Main Documentation (Sticky Note)
- **Type / role:** Sticky Note (non-executing documentation).
- **Content highlights:**
  - Explains trigger → lookup → validate → encrypt (AES‑128) → Gmail delivery.
  - States that failures route to Slack and are logged for auditing (note: no explicit logging node exists in the provided workflow).
  - Setup steps for Employee DB, Drive folder selection, encryption password mapping, Gmail sender, Slack alerts.
- **Potential mismatch to actual flow:**
  - “logs the failure for auditing” is not implemented in nodes.
  - “password dynamically mapped” not visible in encryption node parameters.
  - “missing password routes to error-handling path” is only true if errors are handled; the Code node currently throws without an explicit branch.

#### Node: Security Overview (Sticky Note)
- **Type / role:** Sticky Note (non-executing).
- **Content highlights:** Indicates encryption lifecycle and routing focus for grouped nodes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger: Watch Secure Inbox | Google Drive | Watches a Drive location for new files to start processing | — | Download PDF Binary | ## How it works<br>This enterprise workflow manages secure payroll distribution. It triggers on new file arrivals, performs a lookup for employee-specific passwords, and validates security metadata. The PDF is encrypted using 128-bit AES. If successful, it is delivered via Gmail. If any step fails (missing password or encryption error), the workflow routes to an error-handling path that alerts admins via Slack and logs the failure for auditing.<br><br>## Setup steps<br>1. **Employee DB**: Connect a Google Sheet containing Employee Emails and National IDs.<br>2. **Drive**: Select your source folder for unencrypted payslips.<br>3. **Encryption**: Connect your HTML to PDF node; the password is dynamically mapped to the 'National ID' column.<br>4. **Gmail**: Set up your payroll sender account.<br>5. **Error Handling**: Configure the Slack channel to receive alerts for any failed encryptions. |
| Download PDF Binary | Google Drive | Downloads the triggered PDF as binary | Trigger: Watch Secure Inbox | Lookup Employee Password | (same as above) |
| Lookup Employee Password | Google Sheets | Looks up employee email + National ID in Sheets | Download PDF Binary | Pre-Encryption Integrity Check | ### 🛡️ Secure Processing & Fail-Safe Routing<br>Nodes grouped here manage the encryption lifecycle and ensure that only successfully locked files are dispatched. |
| Pre-Encryption Integrity Check | Code | Validates security metadata exists; throws if missing | Lookup Employee Password | Encrypt PDF (AES-128) | ### 🛡️ Secure Processing & Fail-Safe Routing<br>Nodes grouped here manage the encryption lifecycle and ensure that only successfully locked files are dispatched. |
| Encrypt PDF (AES-128) | HTMLCSS to PDF (pdfManipulation/encrypt) | Encrypts the PDF (AES-128) and continues on error output | Pre-Encryption Integrity Check | IF: Encryption Success? | ### 🛡️ Secure Processing & Fail-Safe Routing<br>Nodes grouped here manage the encryption lifecycle and ensure that only successfully locked files are dispatched. |
| IF: Encryption Success? | If | Routes success vs failure based on encryption output | Encrypt PDF (AES-128) | Deliver Secure Email; Alert Admin (Slack) | ### 🛡️ Secure Processing & Fail-Safe Routing<br>Nodes grouped here manage the encryption lifecycle and ensure that only successfully locked files are dispatched. |
| Deliver Secure Email | Gmail | Sends encrypted payslip to employee | IF: Encryption Success? (true) | — |  |
| Alert Admin (Slack) | Slack | Notifies admins on failure | IF: Encryption Success? (false) | — |  |
| Main Documentation | Sticky Note | Embedded operational documentation | — | — |  |
| Security Overview | Sticky Note | Embedded security grouping note | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**
   1. Add node: **Google Drive**
   2. Configure it as a **watch/trigger** for new files in a folder (scope).
   3. Set:
      - Drive: **My Drive**
      - Folder: choose the “secure inbox” folder (avoid Root in production).
   4. Add/assign **Google Drive OAuth2** credentials.

2. **Download the Incoming PDF**
   1. Add node: **Google Drive**
   2. Operation: **Download**
   3. File ID: expression `{{$json.id}}`
   4. Connect: **Trigger → Download**
   5. Ensure the node outputs a binary property (note its binary key name for later attachment/encryption).

3. **Look Up Employee Record in Google Sheets**
   1. Add node: **Google Sheets**
   2. Operation: **Lookup**
   3. Document ID: set to your employee database spreadsheet ID (replaces `YOUR_EMPLOYEE_DB_ID`)
   4. Configure the lookup key to match how you identify the employee from the file (not shown in JSON). Common patterns:
      - Filename contains employee email/ID; parse it then lookup by that column.
      - Drive file metadata contains an identifier.
   5. Ensure your sheet contains at least:
      - `Email`
      - `National_ID`
   6. Add/assign **Google Sheets OAuth2** credentials.
   7. Connect: **Download → Sheets Lookup**

4. **Validate Security Metadata**
   1. Add node: **Code**
   2. Paste logic equivalent to:
      - If `National_ID` missing, throw error.
      - Otherwise return the first item.
   3. Connect: **Sheets Lookup → Code**
   4. Recommended hardening (optional but important):
      - If no input items, throw a clear “Employee not found” error.
      - Avoid dropping items if you expect batches.

5. **Encrypt the PDF**
   1. Add node: **HTMLCSS to PDF** (the node used is `n8n-nodes-htmlcsstopdf`)
   2. Resource: **PDF Manipulation**
   3. Operation: **Encrypt**
   4. Enable: **Continue on fail / continue error output** (so failures can be routed)
   5. Configure:
      - Input binary property: set to the downloaded PDF binary key
      - Password: map to `{{$node["Lookup Employee Password"].json.National_ID}}` (or the current item if you merge data)
      - Encryption strength: **AES-128** (as intended by workflow naming)
   6. Add/assign the HTMLCSS to PDF API credentials.
   7. Connect: **Code → Encrypt**

6. **Add Success/Failure Branching**
   1. Add node: **IF**
   2. Condition (boolean): `!!$json.data` (or update to match the encryption node’s real success output schema)
   3. Connect: **Encrypt → IF**

7. **Send Email via Gmail (Success Path)**
   1. Add node: **Gmail**
   2. Operation: **Send**
   3. To: `{{$node["Lookup Employee Password"].json.Email}}`
   4. Subject: `Secure Payslip: {{$now.minus({ months: 1 }).toFormat('MMMM yyyy')}}`
   5. Message: include unlock instructions.
   6. Attach the encrypted PDF:
      - Configure Gmail node attachments to use the binary output of the encryption node (exact field depends on node output; map accordingly).
   7. Add/assign **Gmail OAuth2** credentials.
   8. Connect: **IF (true) → Gmail**

8. **Send Slack Alert (Failure Path)**
   1. Add node: **Slack**
   2. Operation: **Post message** (to a channel is usually safer than “user” unless you set a user explicitly)
   3. Text: include employee email and failure reason.
   4. Configure destination (channel or user) explicitly.
   5. Add/assign **Slack OAuth2** credentials.
   6. Connect: **IF (false) → Slack**

9. **(Recommended) Align “metadata missing” with Slack routing**
   - To ensure missing National ID also alerts Slack, either:
     - Change the Code node to *not throw* and instead output a failure object that routes into the same IF/Slack path, or
     - Use n8n’s error workflow / error trigger, or
     - Wrap validation in an IF node and route to Slack directly.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Employee DB: Connect a Google Sheet containing Employee Emails and National IDs.” | Sticky note: Main Documentation |
| “Drive: Select your source folder for unencrypted payslips.” | Sticky note: Main Documentation |
| “Encryption: … password is dynamically mapped to the 'National ID' column.” | Sticky note: Main Documentation (verify mapping is actually configured in the encryption node) |
| “Error Handling: Configure the Slack channel to receive alerts for any failed encryptions.” | Sticky note: Main Documentation (current Slack node destination appears unset) |
| “Nodes grouped here manage the encryption lifecycle and ensure that only successfully locked files are dispatched.” | Sticky note: Security Overview |
| “logs the failure for auditing” | Mentioned in Main Documentation, but no logging/audit node exists in the provided workflow (add Google Sheets log, DB insert, or Drive move/tag to implement). |