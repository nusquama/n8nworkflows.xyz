Send a weekly Sunday digest email from Notion via SMTP

https://n8nworkflows.xyz/workflows/send-a-weekly-sunday-digest-email-from-notion-via-smtp-20951


# Send a weekly Sunday digest email from Notion via SMTP

### 1. Workflow Overview

This workflow is designed to automate a weekly review process by gathering key metrics and action items from four distinct Notion databases—Finance, Projects, Pipeline, and Content—compiling them into a structured plain-text digest, and emailing the summary to a designated recipient. 

The logical execution is organized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Initiates the workflow either on a weekly schedule (every Sunday at 09:00) or via manual execution, followed by setting global configuration parameters (database IDs, notification email, and threshold values).
- **1.2 Notion Data Retrieval:** Sequentially queries four Notion databases to fetch overdue invoices, active projects, sales pipeline deals matching specific staleness criteria, and content items in draft or scheduled states.
- **1.3 Compilation and Delivery:** Processes the retrieved datasets using JavaScript to format a plain-text summary and dispatches the compiled email via SMTP.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** Acts as the entry point for the workflow, supporting both automated weekly runs and manual testing, and initializes environmental variables required by downstream nodes.
- **Nodes Involved:** 
  - `Every Sunday at 09:00`
  - `Run manually to test`
  - `Set your settings`
- **Node Details:**
  - **Every Sunday at 09:00**
    - *Type & Role:* `Schedule Trigger` node. Triggers the workflow execution based on a cron expression.
    - *Configuration:* Cron expression set to `0 9 * * 0` (Every Sunday at 09:00, with workflow timezone set to `Europe/Oslo`).
    - *Connections:* Output connects to `Set your settings`.
    - *Potential Failures:* Timezone misconfigurations may cause execution at unintended local times.
  - **Run manually to test**
    - *Type & Role:* `Manual Trigger` node. Allows on-demand execution for testing and debugging.
    - *Configuration:* Default parameters.
    - *Connections:* Output connects to `Set your settings`.
  - **Set your settings**
    - *Type & Role:* `Set` (Edit Fields) node. Defines static configuration parameters for the workflow.
    - *Configuration:* Manual assignment mode assigning variables: `NOTIFY_EMAIL`, `FINANCE_DB_ID`, `PROJECTS_DB_ID`, `PIPELINE_DB_ID`, `CONTENT_DB_ID`, and `COLD_AFTER_DAYS` (set to `7`).
    - *Connections:* Input from triggers; output connects to `Get overdue invoices from Notion`.
    - *Potential Failures:* Incorrect or placeholder database IDs will cause subsequent Notion API calls to fail.

#### Block 1.2: Notion Data Retrieval
- **Overview:** Sequentially queries four Notion databases to retrieve relevant pages using manual filters or raw data extraction for post-filtering.
- **Nodes Involved:**
  - `Get overdue invoices from Notion`
  - `Get active projects from Notion`
  - `Get sales pipeline from Notion`
  - `Get content plan from Notion`
- **Node Details:**
  - **Get overdue invoices from Notion**
    - *Type & Role:* `Notion` node. Fetches database pages representing overdue invoices.
    - *Configuration:* Resource `databasePage`, operation `getAll`, manual filter where `Status` (select) equals `Overdue`. Database ID dynamically retrieved via expression: `={{ $('Set your settings').first().json.FINANCE_DB_ID }}`. `Always Output Data` enabled.
    - *Connections:* Input from `Set your settings`; output connects to `Get active projects from Notion`.
    - *Potential Failures:* Invalid Notion API credentials, mismatched database schema property names (`Status`), or missing database sharing permissions with the integration.
  - **Get active projects from Notion**
    - *Type & Role:* `Notion` node. Fetches active projects from the project database.
    - *Configuration:* Resource `databasePage`, operation `getAll`, manual filter where `Status` (select) equals `Active`. Database ID retrieved via expression: `={{ $('Set your settings').first().json.PROJECTS_DB_ID }}`. `Always Output Data` enabled.
    - *Connections:* Input from `Get overdue invoices from Notion`; output connects to `Get sales pipeline from Notion`.
    - *Potential Failures:* Notion API rate limits or property naming discrepancies (`Status`).
  - **Get sales pipeline from Notion**
    - *Type & Role:* `Notion` node. Retrieves all records from the sales pipeline database for programmatic filtering.
    - *Configuration:* Resource `databasePage`, operation `getAll`. Database ID retrieved via expression: `={{ $('Set your settings').first().json.PIPELINE_DB_ID }}`. `Always Output Data` enabled.
    - *Connections:* Input from `Get active projects from Notion`; output connects to `Get content plan from Notion`.
    - *Potential Failures:* Large databases may experience pagination issues if limits are exceeded, though `returnAll` is enabled.
  - **Get content plan from Notion**
    - *Type & Role:* `Notion` node. Retrieves all records from the content plan database for programmatic filtering.
    - *Configuration:* Resource `databasePage`, operation `getAll`. Database ID retrieved via expression: `={{ $('Set your settings').first().json.CONTENT_DB_ID }}`. `Always Output Data` enabled.
    - *Connections:* Input from `Get sales pipeline from Notion`; output connects to `Build Sunday digest text`.
    - *Potential Failures:* API connectivity errors or missing integration access.

#### Block 1.3: Compilation and Delivery
- **Overview:** Transforms the raw Notion payloads into a clean plain-text digest and transmits it via SMTP email.
- **Nodes Involved:**
  - `Build Sunday digest text`
  - `Send Sunday digest email`
- **Node Details:**
  - **Build Sunday digest text**
    - *Type & Role:* `Code` node (JavaScript). Processes datasets from all upstream Notion nodes, filters pipeline and content records based on custom business logic, and constructs the final email body and subject.
    - *Configuration:* JavaScript execution parsing node outputs, extracting Notion title properties, evaluating deal staleness based on `COLD_AFTER_DAYS`, and assembling bulleted sections.
    - *Key Expressions/Variables:* Uses `$('Node Name').all()` to aggregate items across nodes and references configuration properties.
    - *Connections:* Input from `Get content plan from Notion`; output connects to `Send Sunday digest email`.
    - *Potential Failures:* JavaScript execution errors if unexpected data structures or null property references are returned from Notion.
  - **Send Sunday digest email**
    - *Type & Role:* `Email Read/Send` (SMTP) node. Sends the compiled plain-text digest to the recipient.
    - *Configuration:* Email format set to text. Subject mapped via expression `={{ $json.subject }}`, text body mapped via `={{ $json.body }}`, recipient and sender email mapped via `={{ $('Set your settings').first().json.NOTIFY_EMAIL }}`. Attribution appending disabled.
    - *Connections:* Input from `Build Sunday digest text`.
    - *Potential Failures:* SMTP authentication failures, incorrect port/host settings, or network restrictions blocking outbound mail traffic.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - Overview` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## Sunday digest from your Notion workspace<br><br>Every Sunday morning this workflow reads four Notion databases and emails you one plain-text summary, so you start the week knowing what needs attention.<br><br>**Who it's for:** freelancers, consultants and solo founders who run clients, invoices, sales and content in Notion.<br><br>### How it works<br>1. A schedule trigger runs every Sunday at 09:00 (cron `0 9 * * 0`, workflow timezone Europe/Oslo).<br>2. Four Notion nodes fetch overdue invoices, active projects, your sales pipeline and your content plan.<br>3. A Code node keeps deals in Contacted, Proposal or Negotiation that haven't been touched for 7+ days, plus content in Draft or Scheduled, and builds the email text.<br>4. One email goes to the address you set.<br><br>### Setup steps<br>1. Connect your Notion credential in each of the four Notion nodes, and share the databases with that integration.<br>2. Open **Set your settings** and paste your four database IDs and your email address.<br>3. Connect an SMTP credential in **Send Sunday digest email** (or swap in the Gmail/Outlook node).<br>4. Change the schedule in **Every Sunday at 09:00**, and the timezone under Workflow settings.<br>5. Click **Run manually to test**, check your inbox, then activate.<br><br>### Customization<br>- Property names used: `Name`, `Status`, `Amount`, `Stage`, `Last touched`, `Deal name`, `Title`. Rename them in the Notion filters and the Code node to match your databases.<br>- Change `COLD_AFTER_DAYS` to decide when a deal counts as cold.<br>- Add or remove sections in the Code node. |
| `Sticky Note - Section 1` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 1. Schedule<br>Runs every Sunday at 09:00. Use the manual trigger to test. |
| `Sticky Note - Section 2` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 2. Settings<br>Your database IDs, email and cold-deal threshold. |
| `Sticky Note - Section 3` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 3. Read Notion<br>Fetches invoices, projects, pipeline and content. |
| `Sticky Note - Section 4` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 4. Build and send<br>Turns the results into one email. |
| `Every Sunday at 09:00` | `n8n-nodes-base.scheduleTrigger` | Weekly automated trigger | None | `Set your settings` | ## 1. Schedule<br>Runs every Sunday at 09:00. Use the manual trigger to test. |
| `Run manually to test` | `n8n-nodes-base.manualTrigger` | Manual test trigger | None | `Set your settings` | ## 1. Schedule<br>Runs every Sunday at 09:00. Use the manual trigger to test. |
| `Set your settings` | `n8n-nodes-base.set` | Configuration storage | `Every Sunday at 09:00`, `Run manually to test` | `Get overdue invoices from Notion` | ## 2. Settings<br>Your database IDs, email and cold-deal threshold. |
| `Get overdue invoices from Notion` | `n8n-nodes-base.notion` | Fetch overdue invoices | `Set your settings` | `Get active projects from Notion` | ## 3. Read Notion<br>Fetches invoices, projects, pipeline and content. |
| `Get active projects from Notion` | `n8n-nodes-base.notion` | Fetch active projects | `Get overdue invoices from Notion` | `Get sales pipeline from Notion` | ## 3. Read Notion<br>Fetches invoices, projects, pipeline and content. |
| `Get sales pipeline from Notion` | `n8n-nodes-base.notion` | Fetch sales pipeline | `Get active projects from Notion` | `Get content plan from Notion` | ## 3. Read Notion<br>Fetches invoices, projects, pipeline and content. |
| `Get content plan from Notion` | `n8n-nodes-base.notion` | Fetch content plan | `Get sales pipeline from Notion` | `Build Sunday digest text` | ## 3. Read Notion<br>Fetches invoices, projects, pipeline and content. |
| `Build Sunday digest text` | `n8n-nodes-base.code` | Format email content | `Get content plan from Notion` | `Send Sunday digest email` | ## 4. Build and send<br>Turns the results into one email. |
| `Send Sunday digest email` | `n8n-nodes-base.emailSend` | Send email digest via SMTP | `Build Sunday digest text` | None | ## 4. Build and send<br>Turns the results into one email. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Triggers:**
   - Add a **Schedule Trigger** node named `Every Sunday at 09:00`. Configure the interval rule to use a cron expression: `0 9 * * 0`. Set workflow timezone to `Europe/Oslo` in workflow settings.
   - Add a **Manual Trigger** node named `Run manually to test`.

2. **Create the Configuration Node:**
   - Add a **Set** (Edit Fields) node named `Set your settings`.
   - Configure manual assignments with the following fields:
     - `NOTIFY_EMAIL` (String): `you@example.com`
     - `FINANCE_DB_ID` (String): `PASTE_YOUR_FINANCE_DATABASE_ID`
     - `PROJECTS_DB_ID` (String): `PASTE_YOUR_PROJECTS_DATABASE_ID`
     - `PIPELINE_DB_ID` (String): `PASTE_YOUR_PIPELINE_DATABASE_ID`
     - `CONTENT_DB_ID` (String): `PASTE_YOUR_CONTENT_DATABASE_ID`
     - `COLD_AFTER_DAYS` (Number): `7`
   - Connect both `Every Sunday at 09:00` and `Run manually to test` to this node.

3. **Create the Notion Retrieval Nodes:**
   - Add a **Notion** node named `Get overdue invoices from Notion`.
     - Set Resource to `Database Page` and Operation to `Get All`.
     - Enable manual filters: Condition where `Status` (select) equals `Overdue`.
     - Set Database ID using the expression: `={{ $('Set your settings').first().json.FINANCE_DB_ID }}`.
     - Enable `Always Output Data` and set `Execute Once` to true.
     - Connect `Set your settings` output to this node.
   - Add a **Notion** node named `Get active projects from Notion`.
     - Set Resource to `Database Page` and Operation to `Get All`.
     - Enable manual filters: Condition where `Status` (select) equals `Active`.
     - Set Database ID using the expression: `={{ $('Set your settings').first().json.PROJECTS_DB_ID }}`.
     - Enable `Always Output Data` and set `Execute Once` to true.
     - Connect `Get overdue invoices from Notion` to this node.
   - Add a **Notion** node named `Get sales pipeline from Notion`.
     - Set Resource to `Database Page` and Operation to `Get All`.
     - Set Database ID using the expression: `={{ $('Set your settings').first().json.PIPELINE_DB_ID }}`.
     - Enable `Always Output Data` and set `Execute Once` to true.
     - Connect `Get active projects from Notion` to this node.
   - Add a **Notion** node named `Get content plan from Notion`.
     - Set Resource to `Database Page` and Operation to `Get All`.
     - Set Database ID using the expression: `={{ $('Set your settings').first().json.CONTENT_DB_ID }}`.
     - Enable `Always Output Data` and set `Execute Once` to true.
     - Connect `Get sales pipeline from Notion` to this node.
   - *Credential Requirement:* Configure valid Notion API credentials across all four Notion nodes and ensure target databases are shared with the integration.

4. **Create the Processing Node:**
   - Add a **Code** node named `Build Sunday digest text`.
   - Set Mode to `JavaScript` and insert the aggregation and filtering script that compiles the items into a plain-text email body and subject.
   - Connect `Get content plan from Notion` to this node.

5. **Create the Email Delivery Node:**
   - Add an **Email Read/Send** node named `Send Sunday digest email`.
   - Configure parameters:
     - Email Format: `Text`
     - To Email: `={{ $('Set your settings').first().json.NOTIFY_EMAIL }}`
     - From Email: `={{ $('Set your settings').first().json.NOTIFY_EMAIL }}`
     - Subject: `={{ $json.subject }}`
     - Text: `={{ $json.body }}`
     - Options: Append Attribution set to `false`.
   - *Credential Requirement:* Configure valid SMTP credentials in n8n.
   - Connect `Build Sunday digest text` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow Disclaimer | The provided text originates exclusively from an automated n8n workflow, compliant with content policies. |
| Target Audience | Freelancers, consultants, and solo founders managing clients, invoices, sales, and content via Notion. |