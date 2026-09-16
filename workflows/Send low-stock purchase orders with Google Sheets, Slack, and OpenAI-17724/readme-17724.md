Send low-stock purchase orders with Google Sheets, Slack, and OpenAI

https://n8nworkflows.xyz/workflows/send-low-stock-purchase-orders-with-google-sheets--slack--and-openai-17724


# Send low-stock purchase orders with Google Sheets, Slack, and OpenAI

### 1. Workflow Overview

This workflow automates inventory monitoring and purchase order generation by pulling stock data from Google Sheets, calculating stock percentages and reorder urgency, drafting purchase order notifications using OpenAI, posting alerts to Slack, and updating both the inventory spreadsheet and a dedicated reorder log.

The logic is grouped into three main functional blocks:
- **1.1 Trigger, Fetch & Failsafe:** Handles manual or scheduled workflow initiation, error capturing, initial data retrieval from the inventory sheet, and batch iteration over individual product rows.
- **1.2 Stock Evaluation & Summary:** Computes stock levels, urgency, and estimated reorder costs, updates the audit timestamp, filters products requiring reorders, prevents duplicate daily orders, and posts a final execution summary to Slack.
- **1.3 Purchase Order Generation:** Interfaces with OpenAI to generate professional Slack purchase order messages for low-stock items, posts notifications to Slack, and updates inventory records and reorder logs.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger, Fetch & Failsafe
- **Overview:** Initiates the workflow either manually or via a scheduled trigger, catches global errors (when enabled), fetches all active product inventory records from Google Sheets, and sequences processing row by row.
- **Nodes Involved:** `Manual Trigger`, `Check Every 4 Hours`, `Error Trigger`, `🚨 Slack Error Alert`, `Get Active Products`, `Process One Product at a Time`.
- **Node Details:**
  - **Manual Trigger** (`n8n-nodes-base.manualTrigger`)
    - *Role:* Entry point for on-demand execution.
    - *Configuration:* Default.
    - *Connections:* Output connects to `Get Active Products`.
    - *Failure Modes:* None.
  - **Check Every 4 Hours** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Scheduled alternative entry point.
    - *Configuration:* Cron expression set to `0 */4 * * *` (disabled by default).
    - *Connections:* None active.
    - *Failure Modes:* None.
  - **Error Trigger** (`n8n-nodes-base.errorTrigger`)
    - *Role:* Captures workflow execution failures.
    - *Configuration:* Disabled by default.
    - *Connections:* Output connects to `🚨 Slack Error Alert`.
    - *Failure Modes:* None.
  - **🚨 Slack Error Alert** (`n8n-nodes-base.slack`)
    - *Role:* Notifies team via Slack when a workflow error occurs.
    - *Configuration:* Uses OAuth2 authentication; formats dynamic error messages using execution metadata variables (`{{ $json.workflow.name }}`, `{{ $json.execution.error.node.name }}`, etc.). Disabled by default.
    - *Connections:* Input from `Error Trigger`.
    - *Failure Modes:* Slack API connectivity errors or missing OAuth2 credentials.
  - **Get Active Products** (`n8n-nodes-base.googleSheets`)
    - *Role:* Retrieves inventory rows from Google Sheets.
    - *Configuration:* Operation set to `get` (all rows); specifies document ID and sheet name (`Inventory`).
    - *Connections:* Input from `Manual Trigger`; output connects to `Process One Product at a Time`.
    - *Failure Modes:* Google Sheets API rate limits, authentication expiry, or incorrect sheet/document IDs.
  - **Process One Product at a Time** (`n8n-nodes-base.splitInBatches`)
    - *Role:* Iterates through incoming product lists sequentially (batch size 1).
    - *Configuration:* Options set with `reset: false`.
    - *Connections:* Input from `Get Active Products` and `Log to Reorder Log Sheet` (loopback); output 0 connects to `Slack — Summary Report` (when loop finishes), output 1 connects to `Calculate Stock Level & Urgency`.
    - *Failure Modes:* Infinite loop risks if loopback conditions are misconfigured.

#### 1.2 Stock Evaluation & Summary
- **Overview:** Evaluates stock metrics for each product, logs check timestamps, determines if reorder thresholds are breached, checks for duplicate same-day reorders, and dispatches a summary upon completion.
- **Nodes Involved:** `Calculate Stock Level & Urgency`, `Update Last Checked Date`, `Needs Reorder?`, `Already Reordered Today?`, `Slack — Summary Report`.
- **Node Details:**
  - **Calculate Stock Level & Urgency** (`n8n-nodes-base.code`)
    - *Role:* Computes stock percentage, urgency status, reorder quantity, and estimated costs using custom JavaScript.
    - *Configuration:* Executes inline JS parsing `Current_Stock`, `Max_Stock`, `Reorder_Threshold_Pct`, and `Unit_Price`.
    - *Connections:* Input from `Process One Product at a Time`; output connects to `Update Last Checked Date`.
    - *Failure Modes:* Syntax errors in script or missing/malformed numerical data in sheet columns.
  - **Update Last Checked Date** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates the audit timestamp (`Last_Checked`) for the current product in Google Sheets.
    - *Configuration:* Operation set to `update` using matching column `Product_ID`.
    - *Connections:* Input from `Calculate Stock Level & Urgency`; output connects to `Needs Reorder?`.
    - *Failure Modes:* Sheet write locks, quota limits, or missing matching identifiers.
  - **Needs Reorder?** (`n8n-nodes-base.if`)
    - *Role:* Evaluates whether the calculated stock percentage falls at or below the reorder threshold.
    - *Configuration:* Condition checks if `Needs_Reorder` equals `true`.
    - *Connections:* Input from `Update Last Checked Date`; True output connects to `Already Reordered Today?`, False output loops back to `Process One Product at a Time`.
    - *Failure Modes:* Boolean type mismatches.
  - **Already Reordered Today?** (`n8n-nodes-base.if`)
    - *Role:* Prevents sending multiple purchase orders for the same item on the same calendar day.
    - *Configuration:* Condition compares `Last_Reorder_Date` against today's date (`YYYY-MM-DD`).
    - *Connections:* Input from `Needs Reorder?` (True branch); True output connects to `Generate PO Message`, False output loops back to `Process One Product at a Time`.
    - *Failure Modes:* Timezone discrepancies between n8n execution environment and Google Sheets dates.
  - **Slack — Summary Report** (`n8n-nodes-base.slack`)
    - *Role:* Posts an inventory check completion report to a designated Slack summary channel.
    - *Configuration:* Uses OAuth2 authentication; targets channel `inventory_summary` with formatted markdown text.
    - *Connections:* Input from `Process One Product at a Time` (loop completion branch).
    - *Failure Modes:* Slack API authentication or channel permission issues.

#### 1.3 Purchase Order Generation
- **Overview:** Leverages OpenAI to draft context-aware purchase orders, publishes them to Slack, updates the inventory record's reorder metrics, and logs the transaction.
- **Nodes Involved:** `Generate PO Message`, `OpenAI — PO Model`, `Slack — Send Purchase Order`, `Update Inventory — Reorder Triggered`, `Log to Reorder Log Sheet`.
- **Node Details:**
  - **Generate PO Message** (`@n8n/n8n-nodes-langchain.agent`)
    - *Role:* AI Agent that compiles product and supplier attributes into a professional Slack purchase order message.
    - *Configuration:* Uses system instructions restricting output to concise markdown without backticks or JSON wrapper, constrained to 15 lines.
    - *Connections:* Input from `Already Reordered Today?`; linked to language model `OpenAI — PO Model`; output connects to `Slack — Send Purchase Order`.
    - *Failure Modes:* OpenAI rate limits, context window errors, or prompt compliance failures.
  - **OpenAI — PO Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
    - *Role:* Sub-node providing the underlying LLM for the AI Agent.
    - *Configuration:* Model set to `gpt-4o-mini`.
    - *Connections:* Connected to `Generate PO Message` via AI language model hook.
    - *Failure Modes:* Invalid OpenAI API key, billing quota exhaustion, or API outages.
  - **Slack — Send Purchase Order** (`n8n-nodes-base.slack`)
    - *Role:* Posts the AI-generated purchase order text to the operations Slack channel.
    - *Configuration:* Uses OAuth2 authentication; target channel set to `reorder_1`.
    - *Connections:* Input from `Generate PO Message`; output connects to `Update Inventory — Reorder Triggered`.
    - *Failure Modes:* Missing channel access or token scope restrictions.
  - **Update Inventory — Reorder Triggered** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates inventory row with reorder date, incremented reorder count, and status notes.
    - *Configuration:* Operation set to `update` matching on `Product_ID`; updates `Reorder_Count`, `Last_Reorder_Date`, and `Notes`.
    - *Connections:* Input from `Slack — Send Purchase Order`; output connects to `Log to Reorder Log Sheet`.
    - *Failure Modes:* Sheet sync concurrency issues or unmapped schema columns.
  - **Log to Reorder Log Sheet** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends an audit trail entry of the generated purchase order into a separate logging worksheet.
    - *Configuration:* Operation set to `append` targeting worksheet `Reorder_Log`.
    - *Connections:* Input from `Update Inventory — Reorder Triggered`; output loops back to `Process One Product at a Time`.
    - *Failure Modes:* Schema validation errors or incorrect target sheet GID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| OpenAI — PO Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for PO generation | None (AI Link) | Generate PO Message | 3️⃣ Purchase Order Generation |
| Error Trigger | n8n-nodes-base.errorTrigger | Catches workflow execution errors | None | 🚨 Slack Error Alert | 📦 Real-Time Inventory & Auto-Reorder Pipeline |
| 🚨 Slack Error Alert | n8n-nodes-base.slack | Posts failure alerts to Slack | Error Trigger | None | 📦 Real-Time Inventory & Auto-Reorder Pipeline |
| Slack — Send Purchase Order | n8n-nodes-base.slack | Sends PO message to Slack channel | Generate PO Message | Update Inventory — Reorder Triggered | 3️⃣ Purchase Order Generation |
| Generate PO Message | @n8n/n8n-nodes-langchain.agent | Drafts PO message via AI | Already Reordered Today? | Slack — Send Purchase Order | 3️⃣ Purchase Order Generation |
| Update Inventory — Reorder Triggered | n8n-nodes-base.googleSheets | Updates stock reorder stats | Slack — Send Purchase Order | Log to Reorder Log Sheet | 3️⃣ Purchase Order Generation |
| Log to Reorder Log Sheet | n8n-nodes-base.googleSheets | Appends record to reorder log | Update Inventory — Reorder Triggered | Process One Product at a Time | 3️⃣ Purchase Order Generation |
| Slack — Summary Report | n8n-nodes-base.slack | Posts completion summary to Slack | Process One Product at a Time | None | 2️⃣ Stock Evaluation & Summary |
| Manual Trigger | n8n-nodes-base.manualTrigger | Starts workflow on demand | None | Get Active Products | 📦 Real-Time Inventory & Auto-Reorder Pipeline<br>1️⃣ Trigger, Fetch & Failsafe |
| Check Every 4 Hours | n8n-nodes-base.scheduleTrigger | Scheduled trigger (disabled) | None | None | 📦 Real-Time Inventory & Auto-Reorder Pipeline<br>1️⃣ Trigger, Fetch & Failsafe |
| Process One Product at a Time | n8n-nodes-base.splitInBatches | Iterates product rows sequentially | Get Active Products, Log to Reorder Log Sheet | Slack — Summary Report, Calculate Stock Level & Urgency | 📦 Real-Time Inventory & Auto-Reorder Pipeline<br>1️⃣ Trigger, Fetch & Failsafe |
| Calculate Stock Level & Urgency | n8n-nodes-base.code | Computes stock % and urgency | Process One Product at a Time | Update Last Checked Date | 2️⃣ Stock Evaluation & Summary |
| Update Last Checked Date | n8n-nodes-base.googleSheets | Updates check timestamp in sheet | Calculate Stock Level & Urgency | Needs Reorder? | 2️⃣ Stock Evaluation & Summary |
| Needs Reorder? | n8n-nodes-base.if | Filters low-stock items | Update Last Checked Date | Already Reordered Today?, Process One Product at a Time | 2️⃣ Stock Evaluation & Summary |
| Get Active Products | n8n-nodes-base.googleSheets | Fetches active inventory rows | Manual Trigger | Process One Product at a Time | 📦 Real-Time Inventory & Auto-Reorder Pipeline<br>1️⃣ Trigger, Fetch & Failsafe |
| Already Reordered Today? | n8n-nodes-base.if | Prevents duplicate daily reorders | Needs Reorder? | Generate PO Message, Process One Product at a Time | 2️⃣ Stock Evaluation & Summary |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers and Failsafes:**
   - Add a **Manual Trigger** node.
   - (Optional) Add a **Schedule Trigger** node (`Check Every 4 Hours`), set the cron expression to `0 */4 * * *`, and disable it.
   - (Optional) Add an **Error Trigger** node linked to a **Slack** node (`🚨 Slack Error Alert`) configured with OAuth2 and dynamic error expressions. Disable both if not needed.
2. **Fetch Inventory Data:**
   - Add a **Google Sheets** node named `Get Active Products`. Connect it to the **Manual Trigger**. Configure it to retrieve all rows from your inventory spreadsheet and sheet tab (`Inventory`).
3. **Set Up Batch Processing:**
   - Add a **Split In Batches** node (`Process One Product at a Time`) connected after `Get Active Products`. Set batch size to `1`.
4. **Calculate Stock Metrics:**
   - Add a **Code** node (`Calculate Stock Level & Urgency`) connected to output batch 1 of the batch processor. Insert JavaScript logic to compute `Stock_Pct`, `Needs_Reorder`, `Urgency_Level`, `Reorder_Qty`, and `Estimated_Cost`.
   - Add a **GoogleSheets** node (`Update Last Checked Date`) connected after the Code node. Configure it to update row data matching on `Product_ID` using `Checked_At`.
5. **Evaluate Reorder Conditions:**
   - Add an **If** node (`Needs Reorder?`) checking if `Needs_Reorder` is `true`.
   - Connect the `False` output back to `Process One Product at a Time` to continue the loop.
   - Connect the `True` output to a second **If** node (`Already Reordered Today?`) comparing `Last_Reorder_Date` to today's date (`YYYY-MM-DD`).
   - Connect the `True` (not reordered today) output of this second If node to the AI processing branch. Connect the `False` output back to `Process One Product at a Time`.
6. **Configure AI PO Generation:**
   - Add an **AI Agent** node (`Generate PO Message`). Set the system prompt to instruct the agent to act as an inventory management assistant utilizing Slack mrkdwn formatting.
   - Add an **OpenAI Chat Model** node (`OpenAI — PO Model`), set the model to `gpt-4o-mini`, provide valid OpenAI API credentials, and connect it to the AI Agent via the model input connection.
7. **Send Slack Notifications and Log Results:**
   - Add a **Slack** node (`Slack — Send Purchase Order`) using OAuth2 credentials, targeting your designated reorder channel, with message text set to `={{ $json.output }}`.
   - Add a **Google Sheets** node (`Update Inventory — Reorder Triggered`) to update `Reorder_Count`, `Last_Reorder_Date`, and `Notes`, matching by `Product_ID`.
   - Add a **Google Sheets** node (`Log to Reorder Log Sheet`) set to operation `append`, targeting the `Reorder_Log` sheet tab.
   - Connect the output of `Log to Reorder Log Sheet` back to `Process One Product at a Time` to close the item processing loop.
8. **Configure Completion Summary:**
   - Connect output 0 (loop completion) of `Process One Product at a Time` to a final **Slack** node (`Slack — Summary Report`) configured with OAuth2 to post run confirmation details to your summary channel.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets Template & Structure | Requires columns: `Product_ID`, `Product_Name`, `Category`, `Current_Stock`, `Max_Stock`, `Reorder_Threshold_Pct`, `Supplier_Name`, `Supplier_Contact`, `Reorder_Qty`, `Unit_Price`, `Status`, `Last_Checked`, `Last_Reorder_Date`, `Reorder_Count`, `Notes`. |
| Spreadsheet Source Reference | Linked to Spreadsheet ID `1XXNtTm8_To0chMbZGiyNQ3A0lIWMLHQBruLch1_MpSQ` (Worksheets: `Inventory` and `Reorder_Log`). |