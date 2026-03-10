Review and approve Google Sheets orders by email and notify via LINE

https://n8nworkflows.xyz/workflows/review-and-approve-google-sheets-orders-by-email-and-notify-via-line-13712


# Review and approve Google Sheets orders by email and notify via LINE

## 1. Workflow Overview

This workflow automates an order approval process using Google Sheets as the database, Gmail for manual review requests, and LINE for real-time notifications. It features a "stateless" architecture, meaning approval links contain all necessary tokens to process requests without the workflow needing to maintain an active wait state.

**The logic is divided into two main phases:**
1.  **Order Processing (Section A):** Monitors a Google Sheet for new orders, determines if they require manual approval based on a price threshold, and either auto-approves them or sends a branded HTML email to a reviewer.
2.  **Approval Handling (Section B):** A public webhook receives clicks from the approval email, validates a security token, ensures the order hasn't been processed already, updates the spreadsheet, and notifies the requester via LINE.

---

## 2. Block-by-Block Analysis

### 2.1 Section A: Order Intake
Polls the spreadsheet for new entries and prepares them for routing.
*   **Nodes Involved:** `Google Sheets Trigger`, `New order?`, `Config`, `Prepare Order Data`, `Write Approval Token`, `Needs Approval?`.
*   **Node Details:**
    *   **Google Sheets Trigger:** Polls every minute for new rows. Requires OAuth2 credentials.
    *   **New order? (IF):** Checks if the `status` column is empty to prevent re-processing existing rows.
    *   **Config (Set):** Defines global variables: `approverEmail`, `lineUserId`, `approvalThreshold`, and `n8nWebhookBaseUrl`.
    *   **Prepare Order Data (Set):** Maps sheet columns to internal variables and generates a unique `approvalToken` using a formula: `orderId + timestamp`.
    *   **Write Approval Token (Google Sheets):** Updates the sheet immediately with the generated token to enable future validation.
    *   **Needs Approval? (IF):** Compares the order amount against the `approvalThreshold`.

### 2.2 Email Review Path
Executed when the order amount meets or exceeds the threshold.
*   **Nodes Involved:** `Build Approval Links`, `Mark as Pending Approval`, `Send Approval Request`.
*   **Node Details:**
    *   **Build Approval Links (Set):** Constructs two URLs (Approve/Reject) pointing to the n8n webhook, appending `orderId`, `action`, and `token` as query parameters.
    *   **Mark as Pending Approval (Google Sheets):** Updates the status to `⏳ Pending Approval`.
    *   **Send Approval Request (Gmail):** Sends a styled HTML email with green/red buttons. Includes a `continueOnFail` setting to prevent workflow crashes on SMTP issues.

### 2.3 Auto-Approve Path
Executed when the order amount is below the threshold.
*   **Nodes Involved:** `Mark as Auto-Approved`, `Prepare LINE Notification (Auto)`, `LINE Push – Auto Approved`.
*   **Node Details:**
    *   **Mark as Auto-Approved (Google Sheets):** Updates the status to `✅ Auto-Approved`.
    *   **LINE Push – Auto Approved (HTTP Request):** Sends a POST request to the LINE Messaging API. Requires a Bearer Token.

### 2.4 Section B: Approval Handler
Handles the interaction when a user clicks a link in their email.
*   **Nodes Involved:** `Webhook – Approval Handler`, `Extract Approval Params`, `Read Order from Sheets`, `Token Valid?`, `Already Processed?`, `Approve or Reject?`.
*   **Node Details:**
    *   **Webhook – Approval Handler:** Listens on the path `/order-approval`. Set to "Response Node" mode to show custom HTML to the user.
    *   **Read Order from Sheets:** Performs a lookup using the `orderId` from the URL to fetch the current status and the token stored in the database.
    *   **Token Valid? (IF):** Compares the token in the URL with the token in the Google Sheet. If they don't match, it triggers a 403 Error response.
    *   **Already Processed? (IF):** Checks if the status is still `⏳ Pending Approval`. If not, it informs the user the order was already handled.

### 2.5 Final Actions
The concluding steps for manual approval or rejection.
*   **Nodes Involved:** `Mark as Approved`, `Mark as Rejected`, `LINE Push – Approved`, `LINE Push – Rejected`, `Respond – Order Approved`, `Respond – Order Rejected`.
*   **Node Details:**
    *   **Mark as Approved/Rejected (Google Sheets):** Updates the final status and adds a `reviewedAt` timestamp.
    *   **Respond to Webhook:** Returns a clean HTML page to the browser (e.g., "Order #123 has been approved").

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Google Sheets Trigger | googleSheetsTrigger | Entry Point (Polling) | - | New order? | ⚡ Section A – Order Intake |
| New order? | if | Filter duplicates | Google Sheets Trigger | Config | ⚡ Section A – Order Intake |
| Config | set | Constant setup | New order? | Prepare Order Data | ⚡ Section A – Order Intake |
| Prepare Order Data | set | Data mapping | Config | Write Approval Token | ⚡ Section A – Order Intake |
| Write Approval Token | googleSheets | Update database | Prepare Order Data | Needs Approval? | ⚡ Section A – Order Intake |
| Needs Approval? | if | Logic branching | Write Approval Token | Build Approval Links, Mark as Auto-Approved | ⚡ Section A – Order Intake |
| Build Approval Links | set | URL construction | Needs Approval? | Mark as Pending Approval | 📧 Email Review Path |
| Mark as Pending Approval | googleSheets | Status update | Build Approval Links | Send Approval Request | 📧 Email Review Path |
| Send Approval Request | gmail | Notification | Mark as Pending Approval | - | 📧 Email Review Path |
| Mark as Auto-Approved | googleSheets | Status update | Needs Approval? | Prepare LINE Notification (Auto) | ✅ Auto-Approve Path |
| Prepare LINE Notification (Auto) | set | Message formatting | Mark as Auto-Approved | LINE Push – Auto Approved | ✅ Auto-Approve Path |
| LINE Push – Auto Approved | httpRequest | Notification | Prepare LINE Notification (Auto) | - | ✅ Auto-Approve Path |
| Webhook – Approval Handler | webhook | Entry Point (URL) | - | Extract Approval Params | 🔗 Section B – Approval Handler |
| Extract Approval Params | set | Query parsing | Webhook – Approval Handler | Read Order from Sheets | 🔗 Section B – Approval Handler |
| Read Order from Sheets | googleSheets | Data verification | Extract Approval Params | Token Valid? | 🔗 Section B – Approval Handler |
| Token Valid? | if | Security check | Read Order from Sheets | Already Processed?, Respond – Invalid Token | 🔗 Section B – Approval Handler |
| Already Processed? | if | Idempotency check | Token Valid? | Respond – Already Processed, Approve or Reject? | 🔗 Section B – Approval Handler |
| Approve or Reject? | if | Final route | Already Processed? | Mark as Approved, Mark as Rejected | 🔗 Section B – Approval Handler |
| Mark as Approved | googleSheets | Final update | Approve or Reject? | Prepare LINE Notification (Approved) | 🏁 Approval Actions |
| Prepare LINE Notification (Approved) | set | Message formatting | Mark as Approved | LINE Push – Approved | 🏁 Approval Actions |
| LINE Push – Approved | httpRequest | Notification | Prepare LINE Notification (Approved) | Respond – Order Approved | 🏁 Approval Actions |
| Respond – Order Approved | respondToWebhook | User Feedback | LINE Push – Approved | - | 🏁 Approval Actions |
| Mark as Rejected | googleSheets | Final update | Approve or Reject? | Prepare LINE Notification (Rejected) | 🏁 Approval Actions |
| Prepare LINE Notification (Rejected) | set | Message formatting | Mark as Rejected | LINE Push – Rejected | 🏁 Approval Actions |
| LINE Push – Rejected | httpRequest | Notification | Prepare LINE Notification (Rejected) | Respond – Order Rejected | 🏁 Approval Actions |
| Respond – Order Rejected | respondToWebhook | User Feedback | LINE Push – Rejected | - | 🏁 Approval Actions |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
1.  **Google Sheet:** Create a sheet with headers: `orderId`, `customerName`, `amount`, `approvalToken`, `status`, `reviewedAt`.
2.  **Credentials:**
    *   **Google Sheets:** Service Account (for updates) and OAuth2 (for trigger).
    *   **Gmail:** OAuth2.
    *   **LINE:** HTTP Bearer Auth using your Channel Access Token.

### Step-by-Step Setup
1.  **Trigger:** Add **Google Sheets Trigger**. Set document and sheet. Set polling to `Every Minute`.
2.  **Filter:** Add an **IF** node (`New order?`). Set condition: `status` is empty.
3.  **Config:** Add a **Set** node. Define `approverEmail`, `lineUserId`, `approvalThreshold` (number), and `n8nWebhookBaseUrl`.
4.  **Token Generation:** Add a **Set** node (`Prepare Order Data`). Create `approvalToken` using: `{{ $json.orderId }}-{{ $now.toMillis() }}`.
5.  **Database Sync:** Add **Google Sheets** node (`Write Approval Token`). Operation: `Update`. Key: `orderId`. Columns: `approvalToken`.
6.  **Branching:** Add an **IF** node (`Needs Approval?`). Condition: `amount` >= `approvalThreshold`.
7.  **Auto Path (False branch):** 
    *   Update GS Status to `✅ Auto-Approved`.
    *   Add **HTTP Request** to LINE API (`https://api.line.me/v2/bot/message/push`). Body: `{ "to": "...", "messages": [{"type": "text", "text": "..."}] }`.
8.  **Email Path (True branch):**
    *   Add **Set** node to build `approveUrl` and `rejectUrl`. Format: `BASE_URL/order-approval?orderId=ID&action=approve&token=TOKEN`.
    *   Update GS Status to `⏳ Pending Approval`.
    *   Add **Gmail** node. Set body to HTML and use the link variables for buttons.
9.  **Handler Webhook:** Add a **Webhook** node. Path: `order-approval`. HTTP Method: `GET`. Response Mode: `When Last Node Finishes`.
10. **Validation:** 
    *   Add **Google Sheets** node (`Read Order`). Lookup by `orderId` from webhook query.
    *   Add **IF** node. Compare query `token` with sheet `approvalToken`.
    *   Add **IF** node. Ensure sheet `status` equals `⏳ Pending Approval`.
11. **Finalize:**
    *   Create two paths from an **IF** node (Action equals `approve`).
    *   Each path updates the GS status, sends a LINE HTTP Request, and ends with a **Respond to Webhook** node containing HTML success/fail messages.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Tutorial** | Included in the "Overview" sticky note within the workflow. |
| **Recommended Credentials** | Mix of OAuth2 (Trigger/Gmail) and Service Account (Updates) for stability. |
| **Content Automation Bundle** | [n8n Content Automation Bundle](https://jasonchuang0818.gumroad.com/l/n8n-content-automation-bundle) |