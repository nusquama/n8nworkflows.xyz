Daily Ad Spend Monitoring with Google Sheets and Slack Threshold Alerts

https://n8nworkflows.xyz/workflows/daily-ad-spend-monitoring-with-google-sheets-and-slack-threshold-alerts-6990


# Daily Ad Spend Monitoring with Google Sheets and Slack Threshold Alerts

### 1. Workflow Overview

This workflow automates the daily monitoring of advertising spend data stored in a Google Sheet. Its main purpose is to aggregate daily spend totals, evaluate if the spend exceeds a defined threshold ($100), and send an alert to a Slack channel if the limit is surpassed. It supports both scheduled daily execution and manual triggering for on-demand checks.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Input**: Initiates the workflow either on a scheduled daily basis or manually.
- **1.2 Data Retrieval**: Fetches all advertising spend data rows from a specified Google Sheet.
- **1.3 Data Aggregation and Sorting**: Groups the data by date, sums spend values, sorts the dates descending, and extracts the most recent day’s spend.
- **1.4 Threshold Evaluation and Notification**: Checks if the spend exceeds $100; if yes, sends a Slack notification; otherwise, ends quietly.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger Input

**Overview:**  
This block provides two entry points that kick off the workflow: a scheduled trigger that runs automatically at a regular interval (daily by default), and a manual trigger for testing or on-demand runs.

**Nodes Involved:**  
- `Schedule Workflow`  
- `Test Workflow`

**Node Details:**

- **Schedule Workflow**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers workflow daily based on a cron-like schedule (interval configured to daily).  
  - *Configuration:* Uses n8n’s interval scheduling with default daily execution.  
  - *Connections:* Outputs to `Get Data` node.  
  - *Failure Modes:* Misconfiguration of cron schedule, n8n scheduler issues.

- **Test Workflow**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual start of the workflow for testing or immediate execution.  
  - *Configuration:* No parameters.  
  - *Connections:* Outputs to `Get Data` node.  
  - *Failure Modes:* None typically; manual intervention required.

---

#### 1.2 Data Retrieval

**Overview:**  
Fetches all rows from a specific Google Sheet that contains advertising spend data, including dates and spend amounts.

**Nodes Involved:**  
- `Get Data`

**Node Details:**

- **Get Data**  
  - *Type:* Google Sheets  
  - *Role:* Reads data from a defined Google Sheets document and sheet tab.  
  - *Configuration:*  
    - Document ID: Google Sheet ID for the marketing data spreadsheet.  
    - Sheet Name: Uses the first sheet (gid=0).  
    - Credentials: Uses OAuth2 credentials connected to a Google account with access to the sheet.  
  - *Key Expressions:* None dynamic; static spreadsheet and sheet IDs.  
  - *Input Connections:* From `Schedule Workflow` or `Test Workflow`.  
  - *Output Connections:* To `Sum spend by Day`.  
  - *Failure Modes:*  
    - Authentication or permission errors if OAuth token invalid or sheet not shared properly.  
    - Google Sheets API limits or network issues.

---

#### 1.3 Data Aggregation and Sorting

**Overview:**  
Processes raw data by grouping rows by date, summing the spend values per day, sorting the aggregated results by date descending, and extracting the latest day’s summary.

**Nodes Involved:**  
- `Sum spend by Day`  
- `Sort Dates Descending`  
- `Keep only Last Day`

**Node Details:**

- **Sum spend by Day**  
  - *Type:* Summarize  
  - *Role:* Groups input data by the `Date` field and sums the `Spend ($)` column.  
  - *Configuration:*  
    - Group by field: `Date`.  
    - Aggregation: Sum of `Spend ($)`.  
  - *Input Connections:* From `Get Data`.  
  - *Output Connections:* To `Sort Dates Descending`.  
  - *Failure Modes:*  
    - Missing or malformed `Date` or `Spend ($)` columns.  
    - Empty data sets.

- **Sort Dates Descending**  
  - *Type:* Code (JavaScript)  
  - *Role:* Sorts the summarized data so that the most recent date appears first.  
  - *Configuration:* Custom JavaScript code provided that sorts by date descending.  
  - *Key Code:*  
    ```js
    const items = $input.all();
    items.sort((a, b) => new Date(b.json.Date) - new Date(a.json.Date));
    return items;
    ```  
  - *Input Connections:* From `Sum spend by Day`.  
  - *Output Connections:* To `Keep only Last Day`.  
  - *Failure Modes:*  
    - Invalid date formats causing parsing errors.  
    - Empty input arrays.

- **Keep only Last Day**  
  - *Type:* Set  
  - *Role:* Selects the first item (most recent day) from sorted results and limits output fields to `Date` and `sum_Spend_($)` for clarity.  
  - *Configuration:*  
    - Executes once (single output).  
    - Sets two fields explicitly: `Date` as string, `sum_Spend_($)` as number.  
  - *Input Connections:* From `Sort Dates Descending`.  
  - *Output Connections:* To `Check if Spend over $100`.  
  - *Failure Modes:*  
    - No data input (empty array).  
    - Incorrect field references.

---

#### 1.4 Threshold Evaluation and Notification

**Overview:**  
Evaluates if the most recent day’s spend exceeds $100. If yes, sends a Slack alert; if not, ends the workflow without action.

**Nodes Involved:**  
- `Check if Spend over $100`  
- `Send Slack Message`  
- `Do Nothign. Under 100`

**Node Details:**

- **Check if Spend over $100**  
  - *Type:* IF  
  - *Role:* Compares the aggregated spend against threshold 100.  
  - *Configuration:*  
    - Condition: `sum_Spend_($) > 100` (numeric greater than).  
  - *Input Connections:* From `Keep only Last Day`.  
  - *Output Connections:*  
    - True: to `Send Slack Message`.  
    - False: to `Do Nothign. Under 100`.  
  - *Failure Modes:*  
    - Missing or malformed numeric field.  
    - Expression evaluation errors.

- **Send Slack Message**  
  - *Type:* Slack  
  - *Role:* Sends a notification message to a specific Slack channel when spend exceeds threshold.  
  - *Configuration:*  
    - Text: Static message "The spend for the most recent day is over $100".  
    - Channel: Selected from configured Slack workspace channels (channel ID `C08T2J84F6C`, named "leads").  
    - Credentials: OAuth2 token with chat:write and channels:read scopes.  
  - *Input Connections:* From `Check if Spend over $100` (true branch).  
  - *Output Connections:* None (ends here).  
  - *Failure Modes:*  
    - Invalid or expired Slack OAuth token.  
    - Bot not invited to channel or insufficient permissions.  
    - Network/API errors.

- **Do Nothign. Under 100**  
  - *Type:* NoOp  
  - *Role:* Ends the workflow silently if spend is below threshold; no action taken.  
  - *Input Connections:* From `Check if Spend over $100` (false branch).  
  - *Output Connections:* None.  
  - *Failure Modes:* None.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                             | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                                                                                                                                                   |
|------------------------|---------------------|--------------------------------------------|------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Test Workflow          | Manual Trigger      | Manual start of workflow                    | —                      | Get Data                  | 1️⃣ Schedule or Manual Trigger - Node: `Schedule Workflow` or `Test Workflow` - Purpose: Either run daily via a cron-like rule or manually trigger the flow.                                                                                                    |
| Schedule Workflow      | Schedule Trigger    | Scheduled daily trigger                      | —                      | Get Data                  | 1️⃣ Schedule or Manual Trigger - Node: `Schedule Workflow` or `Test Workflow` - Purpose: Either run daily via a cron-like rule or manually trigger the flow.                                                                                                    |
| Get Data               | Google Sheets       | Fetches all rows from Google Sheet          | Test Workflow, Schedule Workflow | Sum spend by Day          | 2️⃣ Get Google Sheet Data - Node: `Get Data` - What it does: Fetches all rows from your connected sheet. - Setup: OAuth2 credentials, Google Sheets API enabled, shared sheet access. 📎 Sample Sheet: https://docs.google.com/spreadsheets/d/19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA/edit?usp=sharing |
| Sum spend by Day       | Summarize           | Groups data by Date and sums Spend ($)      | Get Data                | Sort Dates Descending     | 3️⃣ Summarize Spend by Day - Node: `Sum spend by Day` - Groups by `Date` and sums `Spend ($)`; requires header row with these columns.                                                                                                                        |
| Sort Dates Descending  | Code (JavaScript)   | Sorts summarized data by date descending    | Sum spend by Day         | Keep only Last Day        | 4️⃣ Sort by Most Recent Date - Node: `Sort Dates Descending` - Custom JS sorts dates descending.                                                                                                                                                               |
| Keep only Last Day     | Set                 | Keeps only most recent day's summary        | Sort Dates Descending    | Check if Spend over $100  | 5️⃣ Select Top Result - Node: `Keep only Last Day` - Captures top row with `Date` and `sum_Spend_($)`.                                                                                                                                                           |
| Check if Spend over $100 | IF                  | Checks if spend exceeds $100                 | Keep only Last Day       | Send Slack Message, Do Nothign. Under 100 | 6️⃣ Check Spend Threshold - Node: `Check if Spend over $100` - IF node comparing `sum_Spend_($) > 100`.                                                                                                                                                           |
| Send Slack Message     | Slack               | Sends Slack alert if threshold exceeded     | Check if Spend over $100 (true) | —                         | 7️⃣ Send Slack Notification - Node: `Send Slack Message` - Sends alert to Slack channel if spend > $100. Setup: Slack app with chat:write, channels:read scopes; channel must be public or bot invited.                                                         |
| Do Nothign. Under 100  | NoOp                | Ends workflow silently if spend under threshold | Check if Spend over $100 (false) | —                         | 8️⃣ No Action if Under Budget - Node: `Do Nothing. Under 100` - No action path when spend ≤ $100.                                                                                                                                                               |
| Sticky Note            | Sticky Note         | Documentation and instructions              | —                      | —                         | Detailed multi-node documentation for setup and logic steps.                                                                                                                                                                                                   |
| Sticky Note1           | Sticky Note         | Documentation continued                      | —                      | —                         | Detailed multi-node documentation for aggregation, sorting, threshold check, and Slack notification.                                                                                                                                                           |
| Sticky Note2           | Sticky Note         | Documentation continued                      | —                      | —                         | Detailed threshold check and Slack notification instructions.                                                                                                                                                                                                  |
| Sticky Note3           | Sticky Note         | Contact info and help                        | —                      | —                         | Contact info for Robert Breen, Automation Consultant and n8n Expert.                                                                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**

   - Add a **Schedule Trigger** node named `Schedule Workflow`.
     - Set interval to daily (default cron-like daily).
   - Add a **Manual Trigger** node named `Test Workflow`.
   - Connect both nodes’ outputs to the next node `Get Data`.

2. **Create Google Sheets Data Retrieval Node**

   - Add a **Google Sheets** node named `Get Data`.
   - Configure:
     - Document ID: `19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA` (or your own sheet).
     - Sheet Name: `gid=0` (first sheet).
     - Credentials: Create and select a Google OAuth2 credential that has access to the sheet.
       - To create this credential:
         - Go to Google Cloud Console.
         - Create a project and enable the Google Sheets API.
         - Create OAuth2 credentials for a desktop or web app.
         - In n8n, add these credentials and authenticate.
     - Ensure the Google Sheet is shared with the OAuth user email.
   - Connect output to `Sum spend by Day`.

3. **Summarize Spend by Date**

   - Add a **Summarize** node named `Sum spend by Day`.
   - Configure:
     - Fields to split by: `Date`.
     - Fields to summarize: sum of `Spend ($)`.
   - Connect output to `Sort Dates Descending`.

4. **Sort Data by Date Descending**

   - Add a **Code** node named `Sort Dates Descending`.
   - Set language: JavaScript.
   - Paste this code:
     ```js
     const items = $input.all();
     items.sort((a, b) => new Date(b.json.Date) - new Date(a.json.Date));
     return items;
     ```
   - Connect output to `Keep only Last Day`.

5. **Keep Only Most Recent Day**

   - Add a **Set** node named `Keep only Last Day`.
   - Configure:
     - Execute Once: enabled.
     - Fields to set:
       - `Date` (type: string) = `={{ $json.Date }}`
       - `sum_Spend_($)` (type: number) = `={{ $json['sum_Spend_($)'] }}`
   - Connect output to `Check if Spend over $100`.

6. **Check Spend Threshold**

   - Add an **IF** node named `Check if Spend over $100`.
   - Configure condition:
     - Numeric operation: `sum_Spend_($)` > `100`.
   - Connect True output to `Send Slack Message`.
   - Connect False output to `Do Nothign. Under 100`.

7. **Send Slack Notification**

   - Add a **Slack** node named `Send Slack Message`.
   - Configure:
     - Text: `The spend for the most recent day is over $100`
     - Channel: select a channel your Slack app is authorized to post to (e.g., channel ID `C08T2J84F6C` or your own).
     - Credentials: create Slack OAuth2 credentials with `chat:write` and `channels:read` scopes.
       - Setup includes creating a Slack app at https://api.slack.com/apps.
       - Install app to your workspace.
       - Copy OAuth token into n8n credentials.
     - Ensure the bot is invited to the target channel if private.
   - No output connections (ends workflow).

8. **No Operation if Under Threshold**

   - Add a **NoOp** node named `Do Nothign. Under 100`.
   - No configuration needed.
   - No output connections (ends workflow).

9. **Connect All Nodes**

   - From `Schedule Workflow` and `Test Workflow` to `Get Data`.
   - `Get Data` → `Sum spend by Day`
   - `Sum spend by Day` → `Sort Dates Descending`
   - `Sort Dates Descending` → `Keep only Last Day`
   - `Keep only Last Day` → `Check if Spend over $100`
   - `Check if Spend over $100` (true) → `Send Slack Message`
   - `Check if Spend over $100` (false) → `Do Nothign. Under 100`

10. **Add Sticky Notes (Optional but Recommended)**

    - Add sticky notes summarizing each block with instructions and important setup links, as in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| To use the Google Sheets node, enable the Google Sheets API in your Google Cloud project and set up OAuth2 credentials properly. Share your Google Sheet with the OAuth email account.                                        | https://console.cloud.google.com/                                                                            |
| Sample Google Sheet for testing: [Marketing Data Sheet - Copy Me](https://docs.google.com/spreadsheets/d/19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA/edit?usp=sharing)                                                       | Sample spreadsheet used by the workflow.                                                                     |
| Slack app setup requires creating an app with `chat:write` and `channels:read` OAuth scopes, installing it to your workspace, and inviting the bot to the target channel if private.                                       | https://api.slack.com/apps                                                                                     |
| Contact for support or consulting: Robert Breen, Automation Consultant and n8n Expert. Email: robert@ynteractive.com; LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/                                            | Support contact                                                                                               |

---

This documentation enables both advanced users and AI agents to fully understand, reproduce, and effectively maintain the "Daily Ad Spend Monitoring with Google Sheets and Slack Threshold Alerts" workflow.