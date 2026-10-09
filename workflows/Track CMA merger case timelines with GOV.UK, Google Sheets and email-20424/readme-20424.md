Track CMA merger case timelines with GOV.UK, Google Sheets and email

https://n8nworkflows.xyz/workflows/track-cma-merger-case-timelines-with-gov-uk--google-sheets-and-email-20424


# Track CMA merger case timelines with GOV.UK, Google Sheets and email

### 1. Workflow Overview

This workflow automates the tracking of Competition and Markets Authority (CMA) merger case updates from GOV.UK, stores and updates structured milestone records in Google Sheets, and delivers an email digest with an attached standalone HTML report. It targets regulatory analysts, legal teams, or researchers who need to monitor UK merger inquiries without manually checking official pages.

The logical execution is organized into four functional blocks:
- **1.1 Input Reception & Configuration:** Initializes execution manually or via a weekday schedule, setting up the global configuration parameters, spreadsheet targets, recipient details, and case lists.
- **1.2 Data Preparation & GOV.UK Fetching:** Validates configured case paths, transforms them into official GOV.UK API endpoints, and retrieves case content sequentially with error handling.
- **1.3 Milestone Extraction & Database Upsert:** Parses change histories, document publications, tables, and dated text entries from raw JSON payloads, emitting structured rows and upserting them into Google Sheets based on stable row keys.
- **1.4 Report Generation & Notification:** Compiles extracted data into an HTML email summary and a downloadable `.html` timeline report, which is delivered via SMTP (if enabled).

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration
- **Overview:** Initializes the workflow either interactively or on a recurring weekday schedule (Monday–Friday at 08:00 Europe/London time) and injects configuration variables containing the target spreadsheet ID, sheet name, email addresses, and an array of CMA merger cases to track.
- **Nodes Involved:** 
  - `Manual Trigger Initiation`
  - `When Weekday at 8 AM`
  - `Set Email and Sheet Config`

##### Node Details:
- **Manual Trigger Initiation**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Entry point for manual testing.
  - **Configuration Choices:** Default parameters.
  - **Inputs / Outputs:** Input: None | Output: Triggers `Set Email and Sheet Config`.
  - **Edge Cases:** None.

- **When Weekday at 8 AM**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` — Cron-based scheduled entry point.
  - **Configuration Choices:** Cron expression set to `0 8 * * 1-5` (Monday through Friday at 08:00).
  - **Inputs / Outputs:** Input: None | Output: Triggers `Set Email and Sheet Config`.
  - **Edge Cases:** Timezone dependency (defaults to workflow setting `Europe/London`).

- **Set Email and Sheet Config**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Injects static configuration variables into the execution stream.
  - **Configuration Choices:** Assigns string values for `spreadsheetId`, `sheetName`, `senderEmail`, `recipientEmail`, and an array of case objects (`caseName` and `casePath`).
  - **Inputs / Outputs:** Input: `Manual Trigger Initiation` or `When Weekday at 8 AM` | Output: Passes config object to `Prepare API Case Requests`.
  - **Edge Cases:** Unconfigured placeholder values (`YOUR_GOOGLE_SHEET_ID`) will cause downstream validation errors.

---

#### Block 1.2: Data Preparation & GOV.UK Fetching
- **Overview:** Validates input case paths against expected naming conventions, rejects duplicates, formats them into structured API endpoints, and fetches the case contents from GOV.UK using batched HTTP requests.
- **Nodes Involved:** 
  - `Prepare API Case Requests`
  - `Fetch GOV UK Case Details`

##### Node Details:
- **Prepare API Case Requests**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript-based data validator and transformer.
  - **Configuration Choices:** Validates that `cases` is a non-empty array, checks that `spreadsheetId` is configured, ensures case paths match `/cma-cases/[a-z0-9-]+`, eliminates duplicates, and generates `caseKey`, `caseName`, `caseUrl`, and `apiUrl`.
  - **Key Expressions / Variables:** Uses regex `/^\/cma-cases\/[a-z0-9-]+$/` for path validation.
  - **Inputs / Outputs:** Input: `Set Email and Sheet Config` | Output: Array of individual case items sent to `Fetch GOV UK Case Details`.
  - **Edge Cases:** Throws errors if duplicate paths are detected, if paths do not match the required format, or if no cases are specified.

- **Fetch GOV UK Case Details**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Performs HTTP GET requests to retrieve raw JSON content from GOV.UK.
  - **Configuration Choices:** URL set to `={{ $json.apiUrl }}`, timeout set to 20,000ms, batching enabled with batch size 1 and 500ms batch interval. Response handling configured never to throw errors on non-2xx codes (`neverError: true`, `fullResponse: true`). Custom headers include `Accept: application/json` and a specific `User-Agent`.
  - **Inputs / Outputs:** Input: `Prepare API Case Requests` | Output: Raw API responses passed to `Extract Case Milestones`.
  - **Edge Cases:** HTTP failures (e.g., 404 or 500) are caught and handled downstream via `onError: continueRegularOutput`.

---

#### Block 1.3: Milestone Extraction & Database Upsert
- **Overview:** Parses the fetched JSON documents to extract metadata, change histories, document publications, table rows, and dated text entries. It generates deterministic row keys and upserts status and milestone rows into Google Sheets.
- **Nodes Involved:** 
  - `Extract Case Milestones`
  - `Upsert Sheets Timeline Rows`

##### Node Details:
- **Extract Case Milestones**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Complex JavaScript parser transforming raw GOV.UK documents into standardized timeline and status records.
  - **Configuration Choices:** Implements HTML entity decoding, date parsing regexes, URL sanitization, hashing functions for signature creation, and fallback error handling for failed requests.
  - **Inputs / Outputs:** Input: `Fetch GOV UK Case Details` | Output: Array of structured milestone and status records sent to `Upsert Sheets Timeline Rows`.
  - **Edge Cases:** Malformed HTML/JSON bodies or network failure objects trigger an `Error` parse status row containing truncated error messages.

- **Upsert Sheets Timeline Rows**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Writes or updates records in Google Sheets.
  - **Configuration Choices:** Operation set to `appendOrUpdate`, matching on `Row Key`. Dynamically resolves document ID and sheet name from the config node. Column mapping set to auto-map input data.
  - **Inputs / Outputs:** Input: `Extract Case Milestones` | Output: Confirmation of sheet operations passed to `Build Timeline Summary Report`.
  - **Edge Cases:** Authentication failures or missing destination sheet/headers halt execution.

---

#### Block 1.4: Report Generation & Notification
- **Overview:** Aggregates parsed records from the current execution, builds a summary HTML email and a standalone HTML report file, converts the file into binary format, and optionally sends it via SMTP.
- **Nodes Involved:** 
  - `Build Timeline Summary Report`
  - `Format Timeline as HTML`
  - `Email Timeline Digest`

##### Node Details:
- **Build Timeline Summary Report**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Report builder generating email body HTML, plain-text equivalents, and full HTML report documents.
  - **Configuration Choices:** Aggregates all items from `Extract Case Milestones`, sorts milestones chronologically, applies HTML escaping, and generates payload objects containing subject lines, plain text, and HTML strings.
  - **Inputs / Outputs:** Input: `Upsert Sheets Timeline Rows` | Output: Report object passed to `Format Timeline as HTML`.
  - **Edge Cases:** Empty milestone sets are handled gracefully with fallback messaging.

- **Format Timeline as HTML**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Binary data helper converting raw HTML strings into downloadable file attachments.
  - **Configuration Choices:** Uses `this.helpers.prepareBinaryData` to convert `timelineHtml` into a `text/html` binary buffer attached as `cma-timeline-[date].html`.
  - **Inputs / Outputs:** Input: `Build Timeline Summary Report` | Output: JSON payload with attached binary file sent to `Email Timeline Digest`.
  - **Edge Cases:** Memory limits if report sizes become excessively large (unlikely for typical case volumes).

- **Email Timeline Digest**
  - **Type & Technical Role:** `n8n-nodes-base.emailSend` — Sends emails via SMTP with attachments.
  - **Configuration Choices:** Node is **disabled by default** (`disabled: true`). Email format set to `both` (HTML and text), pulls recipient and sender addresses from expressions, attaches the binary file named `timeline`, and disables attribution appending.
  - **Inputs / Outputs:** Input: `Format Timeline as HTML` | Output: Final execution termination.
  - **Edge Cases:** Requires active SMTP credentials and enabling the node for mail delivery. Failure halts workflow execution.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Manual Trigger Initiation` | `n8n-nodes-base.manualTrigger` | Entry point for manual runs | None | `Set Email and Sheet Config` | ## Track CMA case timelines with Google Sheets and email<br><br>### How it works<br>Fetches official GOV.UK case data, extracts dated updates and documents, and upserts timeline rows into Google Sheets. Builds one email summary and a complete HTML timeline attachment. Failed cases remain visible in status rows and the report.<br><br>### Setup steps<br>1. Create a Google Sheet with a **Timeline** tab and the exact headers in the setup note below.<br>2. Open **Set Email and Sheet Config**: set the spreadsheet ID, tab name, sender and recipient. Edit the sample case paths.<br>3. Connect your Google Sheets credential to **Upsert Sheets Timeline Rows**.<br>4. Run manually. Check the sheet, source links, case status rows and the HTML attachment in **Format Timeline as HTML**.<br>5. Connect SMTP credentials and enable **Email Timeline Digest** only when ready to send.<br>6. Set the workflow timezone and activate/publish for weekday 08:00 runs. Default: Europe/London.<br><br>### Customization<br>Add CMA case paths, change the schedule, adjust parsing rules or change the report styling. Two historical cases are included as examples.<br><br>### Requirements<br>Google Sheets access; SMTP only for email delivery. Public GOV.UK requests need no API key. Uses built-in nodes, no AI model or other workflow.<br><br>This template covers CMA case pages only. Planned dates remain marked as planned. Check the official source when using a reported deadline. |
| `When Weekday at 8 AM` | `n8n-nodes-base.scheduleTrigger` | Cron scheduled entry point | None | `Set Email and Sheet Config` | ## 1. Start and configure<br><br>Start manually or at 08:00 Monday–Friday in the workflow timezone. Configure the Google Sheet, recipient and case list once.<br><br>Each case needs `caseName` and `casePath` such as `/cma-cases/amazon-slash-irobot-merger-inquiry`. The preparation step validates paths, rejects duplicate cases and builds official API URLs. |
| `Set Email and Sheet Config` | `n8n-nodes-base.set` | Injects configuration variables | `Manual Trigger Initiation`, `When Weekday at 8 AM` | `Prepare API Case Requests` | ## 1. Start and configure<br><br>Start manually or at 08:00 Monday–Friday in the workflow timezone. Configure the Google Sheet, recipient and case list once.<br><br>Each case needs `caseName` and `casePath` such as `/cma-cases/amazon-slash-irobot-merger-inquiry`. The preparation step validates paths, rejects duplicate cases and builds official API URLs. |
| `Prepare API Case Requests` | `n8n-nodes-base.code` | Validates paths and builds API URLs | `Set Email and Sheet Config` | `Fetch GOV UK Case Details` | ## 1. Start and configure<br><br>Start manually or at 08:00 Monday–Friday in the workflow timezone. Configure the Google Sheet, recipient and case list once.<br><br>Each case needs `caseName` and `casePath` such as `/cma-cases/amazon-slash-irobot-merger-inquiry`. The preparation step validates paths, rejects duplicate cases and builds official API URLs. |
| `Fetch GOV UK Case Details` | `n8n-nodes-base.httpRequest` | Fetches raw GOV.UK JSON content | `Prepare API Case Requests` | `Extract Case Milestones` | ## 2. Fetch and parse cases<br><br>Requests are paced one at a time, 500 ms apart. The parser reads GOV.UK `details.metadata`, `details.change_history`, attachments and dated HTML entries.<br><br>A status row is emitted for every case, including HTTP errors, malformed responses and pages with no extracted dates. Deadline/expected entries are marked Planned. Repeated identical entries are deduplicated within a case. |
| `Extract Case Milestones` | `n8n-nodes-base.code` | Parses milestones and generates row payloads | `Fetch GOV UK Case Details` | `Upsert Sheets Timeline Rows` | ## 2. Fetch and parse cases<br><br>Requests are paced one at a time, 500 ms apart. The parser reads GOV.UK `details.metadata`, `details.change_history`, attachments and dated HTML entries.<br><br>A status row is emitted for every case, including HTTP errors, malformed responses and pages with no extracted dates. Deadline/expected entries are marked Planned. Repeated identical entries are deduplicated within a case. |
| `Upsert Sheets Timeline Rows` | `n8n-nodes-base.googleSheets` | Appends or updates sheet rows | `Extract Case Milestones` | `Build Timeline Summary Report` | ## 3. Update sheet and summarize<br><br>Google Sheets matches on **Row Key**. Identical entries update in place; earlier milestone rows remain available as history.<br><br>After the write succeeds, the report builder reads all parsed rows from this run, groups them by case and lists review flags. Source text is escaped before it is inserted into HTML. |
| `Build Timeline Summary Report` | `n8n-nodes-base.code` | Generates HTML reports and email summaries | `Upsert Sheets Timeline Rows` | `Format Timeline as HTML` | ## 3. Update sheet and summarize<br><br>Google Sheets matches on **Row Key**. Identical entries update in place; earlier milestone rows remain available as history.<br><br>After the write succeeds, the report builder reads all parsed rows from this run, groups them by case and lists review flags. Source text is escaped before it is inserted into HTML. |
| `Format Timeline as HTML` | `n8n-nodes-base.code` | Converts HTML report text into a binary attachment | `Build Timeline Summary Report` | `Email Timeline Digest` | ## 4. Attach and email digest<br><br>**Format Timeline as HTML** creates a downloadable `.html` file containing the full timeline. The email uses a compact HTML/plain-text summary and attaches that file; no JavaScript is required.<br><br>**Email Timeline Digest is disabled by default.** Connect SMTP, replace the example email addresses in Set Email and Sheet Config, and enable it when ready. Publish/activate the workflow after the manual checks pass. |
| `Email Timeline Digest` | `n8n-nodes-base.emailSend` | Sends the email digest via SMTP | `Format Timeline as HTML` | None | ## 4. Attach and email digest<br><br>**Format Timeline as HTML** creates a downloadable `.html` file containing the full timeline. The email uses a compact HTML/plain-text summary and attaches that file; no JavaScript is required.<br><br>**Email Timeline Digest is disabled by default.** Connect SMTP, replace the example email addresses in Set Email and Sheet Config, and enable it when ready. Publish/activate the workflow after the manual checks pass. |
| *Multiple Nodes* | *Various* | Database Schema Reference | *N/A* | *N/A* | ## Create the Timeline tab<br><br>Paste these exact column names into row 1 (one name per column).<br><br>`Row Key` · `Case ID` · `Case Name` · `Regulator` · `Sector` · `Parties` · `Jurisdiction` · `Case Status` · `CMA Case URL` · `Snapshot Date` · `Milestone` · `Milestone Date` · `Planned Date` · `Milestone Type` · `Milestone Category` · `Document Title` · `Source Link` · `Parse Status` · `Fetch Error`<br><br>**Row Key** is required for upsert. Case ID is the stable GOV.UK page slug, not a legal case-reference number. Parties contains the page description.<br><br>### Check your first run<br>- Both sample cases should have a status row.<br>- Failed/empty cases should be visible as Error or Needs review.<br>- Run twice: identical Row Keys should update existing rows.<br>- Inspect the HTML attachment before enabling SMTP.<br><br>### History and failures<br>Milestone rows are retained; removed or changed source content does not delete old sheet rows. The report contains only this run's parsed rows. Stable case-status rows update after recovery.<br><br>Fetch/parse problems are included in the report. Google Sheets or SMTP failures stop execution and appear in n8n's execution history. Configure a separate error workflow in n8n settings if you need failure notifications. |

---

### 4. Reproducing the Workflow from Scratch

1. **Set Workflow Timezone:** Open workflow settings and set the timezone to `Europe/London`.
2. **Create Entry Nodes:**
   - Add a `Manual Trigger Initiation` (`n8n-nodes-base.manualTrigger`).
   - Add a `When Weekday at 8 AM` (`n8n-nodes-base.scheduleTrigger`) with a cron expression rule of `0 8 * * 1-5`.
3. **Configure Settings Node:**
   - Add a `Set` node (`n8n-nodes-base.set`) named `Set Email and Sheet Config`.
   - Configure assignments for:
     - `spreadsheetId` (String: your target Google Sheet ID)
     - `sheetName` (String: `Timeline`)
     - `senderEmail` (String: sender address)
     - `recipientEmail` (String: recipient address)
     - `cases` (Array of objects containing `caseName` and `casePath` properties for target CMA cases).
   - Connect both triggers to this node.
4. **Add Case Preparation Node:**
   - Add a `Code` node (`n8n-nodes-base.code`) named `Prepare API Case Requests`.
   - Insert validation logic to verify case paths start with `/cma-cases/`, reject duplicates, and output standardized JSON items containing `caseKey`, `caseName`, `caseUrl`, and `apiUrl`.
   - Connect `Set Email and Sheet Config` to `Prepare API Case Requests`.
5. **Add HTTP Request Node:**
   - Add an `HTTP Request` node (`n8n-nodes-base.httpRequest`) named `Fetch GOV UK Case Details`.
   - Set URL to `={{ $json.apiUrl }}`, timeout to `20000`, batching size to `1` with `500` ms interval.
   - Configure response options: set `neverError: true`, `fullResponse: true`, and `responseFormat: text`.
   - Add headers: `Accept: application/json` and `User-Agent: n8n-CMA-Timeline-Template`.
   - Set error handling (`onError`) to `continueRegularOutput`.
   - Connect `Prepare API Case Requests` to `Fetch GOV UK Case Details`.
6. **Add Milestone Extraction Node:**
   - Add a `Code` node (`n8n-nodes-base.code`) named `Extract Case Milestones`.
   - Insert the parsing JavaScript logic to process raw bodies, extract change history, attachments, table entries, and structured updates, generating status rows and milestone records with unique `Row Key` hashes.
   - Connect `Fetch GOV UK Case Details` to `Extract Case Milestones`.
7. **Add Google Sheets Node:**
   - Add a Google Sheets node (`n8n-nodes-base.googleSheets`) named `Upsert Sheets Timeline Rows`.
   - Set operation to `appendOrUpdate`, matching columns to `Row Key`.
   - Configure Document ID and Sheet Name expressions referencing `Set Email and Sheet Config`.
   - **Credentials:** Connect valid Google Sheets OAuth2/Service Account credentials with edit permissions to the destination sheet (ensure row 1 contains exact headers matching the schema definition).
   - Connect `Extract Case Milestones` to `Upsert Sheets Timeline Rows`.
8. **Add Report Generation Node:**
   - Add a `Code` node (`n8n-nodes-base.code`) named `Build Timeline Summary Report`.
   - Insert report compilation logic to build email subject lines, plain text summaries, email HTML, and standalone report HTML.
   - Connect `Upsert Sheets Timeline Rows` to `Build Timeline Summary Report`.
9. **Add Binary Conversion Node:**
   - Add a `Code` node (`n8n-nodes-base.code`) named `Format Timeline as HTML`.
   - Insert binary conversion logic using `this.helpers.prepareBinaryData` to attach the HTML report as a downloadable file.
   - Connect `Build Timeline Summary Report` to `Format Timeline as HTML`.
10. **Add Email Send Node:**
    - Add an Email Send node (`n8n-nodes-base.emailSend`) named `Email Timeline Digest`.
    - Set the node state to **disabled** initially.
    - Configure parameters: `toEmail`, `fromEmail`, `subject`, `text`, `html`, and options attachments (`timeline`).
    - **Credentials:** Connect SMTP credentials.
    - Connect `Format Timeline as HTML` to `Email Timeline Digest`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| GOV.UK CMA Merger Cases | Public API requests require no API key; limited strictly to GOV.UK `/cma-cases/` paths. |
| Google Sheet Tab Setup | Requires a tab named `Timeline` with exact header columns: `Row Key`, `Case ID`, `Case Name`, `Regulator`, `Sector`, `Parties`, `Jurisdiction`, `Case Status`, `CMA Case URL`, `Snapshot Date`, `Milestone`, `Milestone Date`, `Planned Date`, `Milestone Type`, `Milestone Category`, `Document Title`, `Source Link`, `Parse Status`, `Fetch Error`. |