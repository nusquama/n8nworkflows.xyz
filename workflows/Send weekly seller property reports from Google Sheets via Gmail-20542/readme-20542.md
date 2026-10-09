Send weekly seller property reports from Google Sheets via Gmail

https://n8nworkflows.xyz/workflows/send-weekly-seller-property-reports-from-google-sheets-via-gmail-20542


# Send weekly seller property reports from Google Sheets via Gmail

### 1. Workflow Overview

This workflow automates the generation and distribution of weekly performance reports for property sellers. Operating on a scheduled trigger every Monday morning, it extracts real estate data from a centralized Google Sheets document, aggregates activity metrics (enquiries, viewings, feedback ratings, and market duration) per seller, compiles tailored HTML email summaries with rule-based recommendations, and logs the communication timestamp back into the spreadsheet.

The workflow logic is categorized into three functional blocks:
- **1.1 Schedule & Configuration Initialization:** Establishes the execution timeline and defines global runtime parameters such as sheet identifiers, agency metadata, and reporting thresholds.
- **1.2 Data Ingestion & Report Compilation:** Sequentially reads active listings, leads, and viewing records, aggregates statistics by seller email, generates actionable insights, and dispatches HTML reports via Gmail while securely BCCing the managing agent.
- **1.3 Audit Logging & State Persistence:** Maps processed listings back to their respective reference IDs and updates the Google Sheets audit trail with the precise report delivery timestamp.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration Initialization
- **Overview:** Triggers the automation workflow on a weekly cadence and establishes the baseline environment variables required for downstream processing.
- **Nodes Involved:** `Every Monday at 8am`, `Config`.

##### Node Details:
- **Every Monday at 8am**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron-based workflow entry point).
  - *Configuration Choices:* Configured to trigger via cron expression `0 8 * * 1`.
  - *Input / Output:* No incoming connections; outputs execution payload to the `Config` node.
  - *Potential Failure Types:* Execution delays during n8n queue saturation.

- **Config**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformation and parameter declaration node).
  - *Configuration Choices:* Stores global constants including `googleSheetId`, `agencyName`, `agentName`, `agentEmail`, `timezone`, `currencySymbol`, `reportDays`, `reportStatuses`, and `staleAfterDays`.
  - *Key Expressions or Variables:* Defines variables accessed globally via `$('Config').first().json.[variableName]`.
  - *Input / Output:* Input from `Every Monday at 8am`; output connected to `Read listings`.
  - *Potential Failure Types:* Missing or malformed parameters causing downstream reference errors.

---

#### 2.2 Data Ingestion & Report Compilation
- **Overview:** Pulls transactional records from Google Sheets, cross-references properties against active seller profiles, computes performance metrics, crafts HTML reports, and transmits them via email.
- **Nodes Involved:** `Read listings`, `Read leads`, `Read viewings`, `Build seller reports`, `Email report to seller`.

##### Node Details:
- **Read listings**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet reader).
  - *Configuration Choices:* Reads the "Listings" sheet using the dynamic document ID resolved from the `Config` node.
  - *Key Expressions or Variables:* `={{ $('Config').first().json.googleSheetId }}`
  - *Input / Output:* Input from `Config`; output connected to `Read leads`.
  - *Potential Failure Types:* API authentication failure, invalid sheet name, or missing document ID.

- **Read leads**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet reader).
  - *Configuration Choices:* Reads the "Leads" tab with `executeOnce` set to true to cache data globally.
  - *Key Expressions or Variables:* `={{ $('Config').first().json.googleSheetId }}`
  - *Input / Output:* Input from `Read listings`; output connected to `Read viewings`.
  - *Potential Failure Types:* Rate limiting from Google Sheets API during heavy read cycles.

- **Read viewings**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet reader).
  - *Configuration Choices:* Reads the "Viewings" tab with `executeOnce` set to true.
  - *Input / Output:* Input from `Read leads`; output connected to `Build seller reports`.
  - *Potential Failure Types:* Network timeout or API quota exhaustion.

- **Build seller reports**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript data processor and HTML template generator).
  - *Configuration Choices:* Custom ES6 script parsing listings, leads, and viewings into structured per-seller objects. Computes metrics (enquiries, viewings, ratings, days on market) and applies rule-based advice algorithms.
  - *Key Expressions or Variables:* Iterates over inputs using `$(node).all()` and executes native JavaScript date manipulation via Luxon (`DateTime`).
  - *Input / Output:* Input from `Read viewings`; output connected to `Email report to seller`.
  - *Potential Failure Types:* Syntax errors in customized text arrays or unhandled null values in date/numeric fields.

- **Email report to seller**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email delivery client).
  - *Configuration Choices:* Sends HTML messages configured with dynamic recipient, subject, and body parameters. Configures BCC headers and custom sender identification.
  - *Key Expressions or Variables:* 
    - Send To: `={{ $json.to }}`
    - Subject: `={{ $json.subject }}`
    - Message: `={{ $json.html }}`
    - BCC: `={{ $('Config').first().json.agentEmail }}`
    - Sender Name: `={{ $('Config').first().json.agencyName }}`
  - *Input / Output:* Input from `Build seller reports`; output connected to `List reported listings`.
  - *Potential Failure Types:* OAuth2 token expiration, recipient rejection, or sending quota limits exceeded.

---

#### 2.3 Audit Logging & State Persistence
- **Overview:** Flattens listing references from the generated reports and updates the corresponding rows in Google Sheets with the current reporting timestamp.
- **Nodes Involved:** `List reported listings`, `Save report date in Google Sheets`.

##### Node Details:
- **List reported listings**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformer).
  - *Configuration Choices:* Extracts property reference IDs from the previous compilation step and generates individual payload items containing the current ISO timestamp.
  - *Key Expressions or Variables:* `$(T.nodes.buildReports).all().flatMap(...)`
  - *Input / Output:* Input from `Email report to seller`; output connected to `Save report date in Google Sheets`.
  - *Potential Failure Types:* Empty listing reference arrays resulting in unhandled payload structures.

- **Save report date in Google Sheets**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet record updater).
  - *Configuration Choices:* Performs an update operation matching rows by `Listing Ref` and updating the `Last Report Sent At` column using raw cell formatting.
  - *Key Expressions or Variables:* 
    - Listing Ref: `={{ $json["Listing Ref"] }}`
    - Last Report Sent At: `={{ $json["Last Report Sent At"] }}`
  - *Input / Output:* Input from `List reported listings`; terminal workflow node.
  - *Potential Failure Types:* Mismatched column headers or missing match columns during update operations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every Monday at 8am | n8n-nodes-base.scheduleTrigger | Cron trigger for weekly execution | None | Config | Send weekly reports to your sellers<br><br>Sellers want to know what is happening with their property. This workflow sends each of them a clear weekly update, without you writing a single email.<br><br>### How it works<br>1. Every Monday at 8am it reads **Listings**, **Leads** and **Viewings**.<br>2. For each active listing it counts enquiries and viewings (this week and in total), upcoming viewings, average viewer rating and days on the market.<br>3. It adds viewer comments and a plain-language recommendation (follow-ups, price review, presentation).<br>4. Each seller receives **one email** covering all of their properties; you are in BCC.<br>5. *Last Report Sent At* is saved on each listing.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, time zone, currency.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Record viewer feedback (rating 1-5 and comment) in **Viewings**.<br><br>### Customization tips<br>- Change `reportDays` for a fortnightly report and edit the rules in **Build seller reports**. |
| Config | n8n-nodes-base.set | Defines global runtime configurations | Every Monday at 8am | Read listings | Send weekly reports to your sellers<br><br>Sellers want to know what is happening with their property. This workflow sends each of them a clear weekly update, without you writing a single email.<br><br>### How it works<br>1. Every Monday at 8am it reads **Listings**, **Leads** and **Viewings**.<br>2. For each active listing it counts enquiries and viewings (this week and in total), upcoming viewings, average viewer rating and days on the market.<br>3. It adds viewer comments and a plain-language recommendation (follow-ups, price review, presentation).<br>4. Each seller receives **one email** covering all of their properties; you are in BCC.<br>5. *Last Report Sent At* is saved on each listing.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, time zone, currency.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Record viewer feedback (rating 1-5 and comment) in **Viewings**.<br><br>### Customization tips<br>- Change `reportDays` for a fortnightly report and edit the rules in **Build seller reports**. |
| Read listings | n8n-nodes-base.googleSheets | Reads property records from Google Sheets | Config | Read leads | ## 1. Load your data<br>Listings, leads and viewings from Google Sheets. |
| Read leads | n8n-nodes-base.googleSheets | Reads prospective buyer leads from Google Sheets | Read listings | Read viewings | ## 1. Load your data<br>Listings, leads and viewings from Google Sheets. |
| Read viewings | n8n-nodes-base.googleSheets | Reads viewing appointments from Google Sheets | Read leads | Build seller reports | ## 1. Load your data<br>Listings, leads and viewings from Google Sheets. |
| Build seller reports | n8n-nodes-base.code | Aggregates data and compiles HTML reports | Read viewings | Email report to seller | ## 2. Build & send<br>One report per seller with stats, feedback and advice. |
| Email report to seller | n8n-nodes-base.gmail | Sends compiled seller reports via email | Build seller reports | List reported listings | ## 2. Build & send<br>One report per seller with stats, feedback and advice. |
| List reported listings | n8n-nodes-base.code | Flattens listing references for batch logging | Email report to seller | Save report date in Google Sheets | ## 3. Log the send<br>Saves Last Report Sent At on each listing. |
| Save report date in Google Sheets | n8n-nodes-base.googleSheets | Updates Google Sheets with reporting timestamps | List reported listings | None | ## 3. Log the send<br>Saves Last Report Sent At on each listing. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node. Set the interval configuration to use a Cron expression: `0 8 * * 1`. Name the node `Every Monday at 8am`.
2. **Create the Configuration Node:**
   - Add a **Set** node named `Config`. Configure string and number assignments for: `googleSheetId`, `agencyName`, `agentName`, `agentEmail`, `timezone`, `currencySymbol`, `reportDays`, `reportStatuses`, and `staleAfterDays`. Connect `Every Monday at 8am` to `Config`.
3. **Configure Data Ingestion Nodes:**
   - Add a **Google Sheets** node named `Read listings`. Set the resource to `Sheet`, operation to `Read`, sheet name to `Listings`, and Document ID to `={{ $('Config').first().json.googleSheetId }}`. Connect `Config` to `Read listings`.
   - Add a second **Google Sheets** node named `Read leads`. Set resource to `Sheet`, operation to `Read`, sheet name to `Leads`, Document ID to `={{ $('Config').first().json.googleSheetId }}`, and enable `Execute Once`. Connect `Read listings` to `Read leads`.
   - Add a third **Google Sheets** node named `Read viewings`. Set resource to `Sheet`, operation to `Read`, sheet name to `Viewings`, Document ID to `={{ $('Config').first().json.googleSheetId }}`, and enable `Execute Once`. Connect `Read leads` to `Read viewings`.
4. **Implement Data Processing and Report Generation:**
   - Add a **Code** node named `Build seller reports`. Paste the JavaScript aggregation and template-generation script (utilizing Luxon `DateTime` for calculations, grouping listings by seller email, evaluating metrics, and building HTML strings). Connect `Read viewings` to this node.
5. **Configure Email Dispatch:**
   - Add a **Gmail** node named `Email report to seller`. Set resource to `Message`, operation to `Send`, email type to `HTML`. Map parameters:
     - Send To: `={{ $json.to }}`
     - Subject: `={{ $json.subject }}`
     - Message: `={{ $json.html }}`
     - Options -> BCC List: `={{ $('Config').first().json.agentEmail }}`
     - Options -> Sender Name: `={{ $('Config').first().json.agencyName }}`
     - Options -> Append Attribution: `false`
   - Configure valid Gmail OAuth2 credentials. Connect `Build seller reports` to this node.
6. **Implement Audit Logging:**
   - Add a **Code** node named `List reported listings`. Paste the transformation script that maps array items from `Build seller reports` into discrete row objects containing `Listing Ref` and `Last Report Sent At`. Connect `Email report to seller` to this node.
   - Add a **Google Sheets** node named `Save report date in Google Sheets`. Set resource to `Sheet`, operation to `Update`, sheet name to `Listings`, Document ID to `={{ $('Config').first().json.googleSheetId }}`, mapping mode to `Define Below`, matching columns to `Listing Ref`. Map `Listing Ref` to `={{ $json["Listing Ref"] }}` and `Last Report Sent At` to `={{ $json["Last Report Sent At"] }}` with cell format set to `RAW`. Connect `List reported listings` to this final node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Real Estate Automation Kit Workflow | Source architecture pattern designed for automated residential property management workflows utilizing Google Workspace and Gmail integrations. |