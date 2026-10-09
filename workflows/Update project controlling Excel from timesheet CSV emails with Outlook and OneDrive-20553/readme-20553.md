Update project controlling Excel from timesheet CSV emails with Outlook and OneDrive

https://n8nworkflows.xyz/workflows/update-project-controlling-excel-from-timesheet-csv-emails-with-outlook-and-onedrive-20553


# Update project controlling Excel from timesheet CSV emails with Outlook and OneDrive

### 1. Workflow Overview

The **Project Controlling Assistant** automates the synchronization of monthly timesheet reports (CSV format) into project-controlling Excel templates sent via email. Operating on a scheduled cadence, the workflow detects incoming project reports, processes and validates quantitative records using the Microsoft Graph API, computes resource deviations against planned figures, updates both detailed and cumulative worksheets, and replies to the sender with a fully reconciled workbook and execution summary.

The workflow logic is partitioned into the following functional blocks:

- **1.1 Input Reception & Validation:** Periodically polls Microsoft Outlook for unread emails with attachments, parses incoming files, filters out the Excel template and timesheet CSV based on filename criteria, and stages the Excel template to Microsoft OneDrive.
- **1.2 Template Structure Analysis & Parsing:** Dynamically detects the dimensions of the Excel template via Microsoft Graph, batch-retrieves cell ranges, and structures planned (“PLAN”) versus actual (“IST”) values for individual resources.
- **1.3 Timesheet Data Aggregation:** Imports the incoming CSV attachment, extracts date and duration metrics, aggregates hours per person and month, and normalizes them into person-days (PD).
- **1.4 Deviation Analysis & Metric Computation:** Correlates structured template values with aggregated actual figures, evaluates budget deviations against configurable thresholds, and assigns traffic-light indicators.
- **1.5 Excel Workbook Updates:** Iterates through individual person rows and total rows to update actual values and formula conditions cell-by-cell in the “Ressourcenverbrauch” sheet, and executes conditional upsert operations on the “Kumulgrundlage” sheet within an active workbook session.
- **1.6 Response Delivery & Cleanup:** Compiles execution metrics and accumulated error logs into a text summary, downloads the updated file from OneDrive, sends an email reply with the processed attachment, and marks the original message as read.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Triggers on a schedule, queries Microsoft Outlook for unread messages with file attachments, validates file naming conventions to separate the project template from the time-tracking CSV, and initializes cloud storage staging.
- **Nodes Involved:** `Schedule Trigger`, `Get many messages`, `Identify Attachments`, `Upload the template to OneDrive`.
- **Node Details:**
  - **Schedule Trigger**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node).
    - *Configuration:* Configured to trigger on a fixed interval of every 1 minute.
    - *Key Expressions:* None.
    - *Input/Output:* Output connected to `Get many messages`.
    - *Errors/Edge Cases:* None inherent; downstream nodes handle execution blocks.
  - **Get many messages**
    - *Type & Technical Role:* `n8n-nodes-base.microsoftOutlook` (Action Node).
    - *Configuration:* Retrieves 1 unread message (`readStatus: unread`, `hasAttachments: true`) and downloads binaries using the prefix `attachment_`. Error output is set to continue on error (`onError: continueErrorOutput`).
    - *Key Expressions:* Filter criteria defined in node parameters.
    - *Input/Output:* Input from `Schedule Trigger`; output to `Identify Attachments`.
    - *Credentials:* Microsoft Outlook OAuth2 API (`Yk7dnrJ8F2QJOKwT`).
    - *Errors/Edge Cases:* Authentication expiration or API rate limits.
  - **Identify Attachments**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* JavaScript execution to scan binary attachments. Searches for keywords (`projektcontrolling`, `template`, `mto`) for the Excel file, and (`elts`, `zeiterfassung`, `cn`, `.csv`) for the timesheet file. Throws an explicit error if either file is missing.
    - *Key Expressions:* Iterates over `Object.entries(attachments)`.
    - *Input/Output:* Input from `Get many messages`; output to `Upload the template to OneDrive` and `Extract Acuaul Values "IST"`.
    - *Errors/Edge Cases:* Throws runtime errors if expected filename keywords are absent or attachment counts mismatch.
  - **Upload the template to OneDrive**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Performs a `PUT` request to upload the identified template binary to OneDrive root with a dynamic filename containing the execution and message IDs. Timeout set to 60,000ms with a maximum of 3 retries.
    - *Key Expressions:* `https://graph.microsoft.com/v1.0/me/drive/root:/ProjectControlling_{{ $execution.id }}_{{ $("Get many messages").first().json.id }}.xlsx:/content`.
    - *Input/Output:* Input from `Identify Attachments`; output to `Find the last line`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Network timeouts, invalid file binary input, or storage quota limits.

#### 2.2 Template Structure Analysis & Parsing
- **Overview:** Inspects the uploaded template's worksheet limits, dynamically calculates the consultant and summary row boundaries, and batch-retrieves range values for processing.
- **Nodes Involved:** `Find the last line`, `Generate Ranges Dynamically`, `Retrieve batch ranges  `, `structure ranges`, `Structure Planned "PLAN"  and Actual "IST"`, `Extract PLAN Values`, `Extract Actual "IST" Values `.
- **Node Details:**
  - **Find the last line**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Queries Microsoft Graph API for the `usedRange` of the `Ressourcenverbrauch` worksheet. Sets the `Prefer: IdType="Id"` header.
    - *KeyExpressions:* `https://graph.microsoft.com/v1.0/me/drive/items/{{ $("Upload the template to OneDrive").first().json.id }}/workbook/worksheets/Ressourcenverbrauch/usedRange`.
    - *Input/Output:* Input from `Upload the template to OneDrive`; output to `Generate Ranges Dynamically`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Worksheet name mismatch (`Ressourcenverbrauch` not found) or file corruption.
  - **Generate Ranges Dynamically**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* JavaScript calculation determining total consultant rows based on row indices and generating dynamic cell range objects (`monat`, `namen`, `plan`, `ist`, `summe_ist`).
    - *Key Expressions:* Parses `usedRange.address` using regex matching.
    - *Input/Output:* Input from `Find the last line`; output to `Retrieve batch ranges  `.
    - *Errors/Edge Cases:* Non-standard template layouts resulting in negative consultant counts.
  - **Retrieve batch ranges  **
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Executes HTTP requests targeting constructed range URLs with `Prefer: IdType="Id"` headers.
    - *Key Expressions:* `={{ $json.url }}`.
    - *Input/Output:* Input from `Generate Ranges Dynamically`; output to `structure ranges`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Graph API batch limitations or malformed range addresses.
  - **structure ranges**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* Maps raw array responses back to defined range types, categorizing month titles, consultant names, plan lines, actual lines, and summary formulas.
    - *Key Expressions:* Array lookups matched against definition lists.
    - *Input/Output:* Input from `Retrieve batch ranges  `; output to `Structure Planned "PLAN"  and Actual "IST"`.
    - *Errors/Edge Cases:* Unmatched range addresses.
  - **Structure Planned "PLAN"  and Actual "IST"**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* Transforms structured rows into discrete items mapped by resource name and monthly periods.
    - *Key Expressions:* Index filtering and mapping loops.
    - *Input/Output:* Input from `structure ranges`; output to `Extract PLAN Values` and `Extract Actual "IST" Values `.
    - *Errors/Edge Cases:* Desynchronized array lengths between names and row sets.
  - **Extract PLAN Values**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* Extracts items where the type property evaluates to `plan`.
    - *Key Expressions:* `items.find(i => i.json.type === 'plan')`.
    - *Input/Output:* Input from `Structure Planned "PLAN"  and Actual "IST"`; output to `Merge Data`.
    - *Errors/Edge Cases:* Missing plan type classification.
  - **Extract Actual "IST" Values **
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* Extracts items where the type property evaluates to `ist`.
    - *Key Expressions:* `items.find(i => i.json.type === 'ist')`.
    - *Input/Output:* Input from `Structure Planned "PLAN"  and Actual "IST"`; output to `Merge Data`.
    - *Errors/Edge Cases:* Missing actual type classification.

#### 2.3 Timesheet Data Extraction & Aggregation
- **Overview:** Imports the CSV timesheet attachment, parses rows containing time registrations, aggregates hours per person and month, and converts them into person-days.
- **Nodes Involved:** `Extract Acuaul Values "IST"`, `Aggregate IST (hours → PD)`.
- **Node Details:**
  - **Extract Acuaul Values "IST"**
    - *Type & Technical Role:* `n8n-nodes-base.extractFromFile` (Data Parser).
    - *Configuration:* Parses CSV file data using a semicolon delimiter (`;`). Binary property name dynamically referenced from attachment identification.
    - *Key Expressions:* `={{ $('Identify Attachments').first().json.istKey }}`.
    - *Input/Output:* Input from `Identify Attachments`; output to `Aggregate IST (hours → PD)`.
    - *Errors/Edge Cases:* Unsupported CSV encoding, delimiter mismatches, or missing columns (`Name`, `Datum`, `Zeit [h]`).
  - **Aggregate IST (hours → PD)**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* JavaScript processing to extract dates (`DD.MM.YYYY`), translate German comma-separated numerical strings into floats, aggregate hours per resource per month, and calculate person-days (`Hours ÷ 8`).
    - *Key Expressions:* String replacements (`replace(',', '.')`) and division scaling.
    - *Input/Output:* Input from `Extract Acuaul Values "IST"`; output to `Merge Data`.
    - *Errors/Edge Cases:* Date parsing failures due to format deviations.

#### 2.4 Deviation Analysis & Metric Computation
- **Overview:** Consolidates plan figures, existing actual figures, and aggregated CSV actuals, executes fuzzy name matching, computes variance, and determines traffic-light alert criteria.
- **Nodes Involved:** `Merge Data`, `Calculate deviations`.
- **Node Details:**
  - **Merge Data**
    - *Type & Technical Role:* `n8n-nodes-base.merge` (Data Combiner).
    - *Configuration:* Combines 3 input streams (Aggregated CSV actuals, Plan values, Template actuals) for comparative calculation.
    - *Key Expressions:* Mode: Multiplex or Combine by position depending on setup.
    - *Input/Output:* Inputs from `Aggregate IST (hours → PD)`, `Extract PLAN Values`, and `Extract Actual "IST" Values `; output to `Calculate deviations`.
    - *Errors/Edge Cases:* Empty input datasets causing data misalignment.
  - **Calculate deviations**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* Core comparison logic. Matches CSV names against template names (full name, last name, first name, compound names). Calculates deviations, applies traffic-light conditions for the reported month ($\text{Green} < 5\%$, $\text{Yellow} < 15\%$, $\text{Red} \ge 15\%$), and prepares cell updates including sum formulas.
    - *Key Expressions:* Custom regex matching and dynamic cell coordinate mapping (`spaltenBuchstabe`).
    - *Input/Output:* Input from `Merge Data`; output to `Enter Actual "IST" Values in the Template`.
    - *Errors/Edge Cases:* Unmatched resource names leading to omitted records.

#### 2.5 Excel Workbook Updates
- **Overview:** Loops through calculated cell updates and summary rows, writes actual values and formulas into the `Ressourcenverbrauch` sheet, and evaluates cumulative sheet requirements via an active Microsoft Graph session.
- **Nodes Involved:** `Enter Actual "IST" Values in the Template`, `Check and Write the Formula`, `Update a Single Cell`, `read Kumulgrundlage `, `Calculate cumulative values`, `Update Kumulgrundlage?`, `Add a new month`, `Find  column`, `Write the month and values`, `Set the new month`, `Update Cumulative Values`, `Merge Values and Close Session`.
- **Node Details:**
  - **Enter Actual "IST" Values in the Template**
    - *Type & Technical Role:* `n8n-nodes-base.splitInBatches` (Flow Control).
    - *Configuration:* Iterates through calculated deviation objects to process cell updates sequentially.
    - *Key Expressions:* Default batch processing.
    - *Input/Output:* Input from `Calculate deviations`; outputs to `read Kumulgrundlage ` (on batch completion or specific triggers) and `Check and Write the Formula`.
    - *Errors/Edge Cases:* High item counts extending execution duration.
  - **Check and Write the Formula**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Evaluates whether the current record is a formula entry or requires value updating based on missing formula indicators.
    - *Key Expressions:* `={{ $json.IstFormel }}` and `={{ $json.FormelFehlt }}`.
    - *Input/Output:* Input from `Enter Actual "IST" Values in the Template`; outputs to `Update a Single Cell` (true branch) and loop continuation (false branch).
    - *Errors/Edge Cases:* Conditional evaluation mismatches.
  - **Update a Single Cell**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Sends a `PATCH` request to Microsoft Graph updating specific cell formulas or numerical values in the `Ressourcenverbrauch` worksheet. Max 3 retries with 2,000ms wait.
    - *Key Expressions:* `https://graph.microsoft.com/v1.0/me/drive/items/{{ $json.FileId }}/workbook/worksheets/Ressourcenverbrauch/range(address='{{ $json.ZelleAdresse }}')`.
    - *Input/Output:* Input from `Check and Write the Formula`; output loops back to `Enter Actual "IST" Values in the Template`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Locked cells, permission errors, or invalid range formats.
  - **read Kumulgrundlage **
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Executes a `GET` request fetching the `usedRange` of the `Kumulgrundlage` worksheet. Configured to execute once.
    - *Key Expressions:* `https://graph.microsoft.com/v1.0/me/drive/items/{{ $("Upload the template to OneDrive").first().json.id }}/workbook/worksheets/Kumulgrundlage/usedRange`.
    - *Input/Output:* Input from `Enter Actual "IST" Values in the Template`; output to `Calculate cumulative values`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Worksheet name mismatch (`Kumulgrundlage` missing).
  - **Calculate cumulative values**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* JavaScript evaluation parsing cumulative rows, checking existence of reporting months, calculating cumulative plan/actual totals, and establishing update flags. Executed once.
    - *Key Expressions:* Array slice operations and cumulative summation logic.
    - *Input/Output:* Input from `read Kumulgrundlage `; output to `Update Kumulgrundlage?`.
    - *Errors/Edge Cases:* Non-standard structure in cumulative data sheets.
  - **Update Kumulgrundlage?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Evaluates whether cumulative values require updating based on `aktualisiereKumul`.
    - *Key Expressions:* `={{ $json.aktualisiereKumul }}`.
    - *Input/Output:* Input from `Calculate cumulative values`; outputs to `Add a new month` (true branch) and `Create  summary` (false branch).
    - *Errors/Edge Cases:* Boolean evaluation errors.
  - **Add a new month**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Initiates an Excel workbook session (`POST /createSession`) with `persistChanges: true` to enable atomic updates. Executed once.
    - *Key Expressions:* `https://graph.microsoft.com/v1.0/me/drive/items/{{ $json.fileId }}/workbook/createSession`.
    - *Input/Output:* Input from `Update Kumulgrundlage?` (true branch); output to `Find  column`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Active session locks or session creation failures.
  - **Find  column**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* Identifies target column indices for new or existing monthly columns in the cumulative worksheet. Executed once.
    - *Key Expressions:* Array mapping of column letters (`A`-`Z`).
    - *Input/Output:* Input from `Add a new month`; output to `Write the month and values`.
    - *Errors/Edge Cases:* Exceeding column capacity limits.
  - **Write the month and values**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Checks if the month already exists to route between creating a new month entry or updating existing values. Executed once.
    - *Key Expressions:* `={{ $json.monatExistiert }}`.
    - *Input/Output:* Input from `Find  column`; outputs to `Set the new month` (new month branch) and `Update Cumulative Values` (existing update branch).
    - *Errors/Edge Cases:* Branch routing misalignment.
  - **Set the new month**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Patches new month labels and cumulative values into the workbook range using the active session ID header. Executed once.
    - *Key Expressions:* Uses session ID header `workbook-session-id`.
    - *Input/Output:* Input from `Write the month and values`; output to `Merge Values and Close Session`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Expired workbook session tokens.
  - **Update Cumulative Values**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Patches updated cumulative values into existing cell ranges using session headers. Executed once.
    - *Key Expressions:* Target range addresses derived from found columns.
    - *Input/Output:* Input from `Write the month and values`; output to `Merge Values and Close Session`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Graph API patch validation errors.
  - **Merge Values and Close Session**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Terminates the active Excel session (`POST /closeSession`) to release workbook locks. Executed once.
    - *Key Expressions:* `https://graph.microsoft.com/v1.0/me/drive/items/{{ $("Find  column").first().json.fileId }}/workbook/closeSession`.
    - *Input/Output:* Inputs from `Set the new month` and `Update Cumulative Values`; output to `Create  summary`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* Session already closed or invalid session ID.

#### 2.6 Response Delivery & Cleanup
- **Overview:** Compiles final metric summaries and accumulated error logs, downloads the finalized template from OneDrive, sends an email reply with the attachment, and marks the original incoming email as read.
- **Nodes Involved:** `Create  summary`, `Download the template from OneDrive`, `Collect errors`, `return Template `, `Mark the email as read`.
- **Node Details:**
  - **Create  summary**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* Generates formatted text summaries including monthly totals, deviations, budget statuses, traffic lights, and cumulative aggregates. Executed once.
    - *Key Expressions:* String concatenation and rounding logic.
    - *Input/Output:* Inputs from `Update Kumulgrundlage?` (false branch) and `Merge Values and Close Session`; output to `Download the template from OneDrive`.
    - *Errors/Edge Cases:* Missing metric inputs.
  - **Download the template from OneDrive**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Cloud Integration).
    - *Configuration:* Downloads the finalized Excel file content as a binary file output named `updated_template`. Executed once.
    - *Key Expressions:* `https://graph.microsoft.com/v1.0/me/drive/items/{{ $("Upload the template to OneDrive").first().json.id }}/content`.
    - *Input/Output:* Input from `Create  summary`; output to `Collect errors`.
    - *Credentials:* Microsoft Drive OAuth2 API (`YVil7uMcVenBBdPV`).
    - *Errors/Edge Cases:* File locks not released by previous session close operations.
  - **Collect errors**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation).
    - *Configuration:* Scans error outputs across preceding nodes, compiles a structured error report if failures occurred, and combines it with the text summary while preserving binary file output. Executed once.
    - *Key Expressions:* Iterates through node executions checking for error properties.
    - *Input/Output:* Input from `Download the template from OneDrive`; output to `return Template `.
    - *Errors/Edge Cases:* Unhandled exception formats.
  - **return Template **
    - *Type & Technical Role:* `n8n-nodes-base.microsoftOutlook` (Action Node).
    - *Configuration:* Sends a reply email to the original sender (`toRecipients` mapped to sender email) with the subject `Projektcontrolling Update`, the summary/error body, and the attached binary file.
    - *Key Expressions:* `={{ $("Get many messages").first().json.from }}`.
    - *Input/Output:* Input from `Collect errors`; output to `Mark the email as read`.
    - *Credentials:* Microsoft Outlook OAuth2 API (`Yk7dnrJ8F2QJOKwT`).
    - *Errors/Edge Cases:* Recipient address invalid or missing attachment binaries.
  - **Mark the email as read**
    - *Type & Technical Role:* `n8n-nodes-base.microsoftOutlook` (Action Node).
    - *Configuration:* Updates the read status of the processed incoming message to `true`. Executed once.
    - *Key Expressions:* `={{ $("Get many messages").first().json.id }}`.
    - *Input/Output:* Input from `return Template `; output none (terminal node).
    - *Credentials:* Microsoft Outlook OAuth2 API (`Yk7dnrJ8F2QJOKwT`).
    - *Errors/Edge Cases:* Message ID not found or already deleted.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation comment | None | None | Checks the inbox every 5 minutes for unread emails with attachments. Expects exactly 2 files:\n- an Excel template (with “projektcontrolling”/“template”/“mto” in the name)\nand\n- an actual time-tracking report as a CSV file (with “elts”/‘zeiterfassung’/“cn” in the name).\n\nThrows an error if either of the two is missing. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation comment | None | None | Uploads the template to OneDrive and dynamically reads the number of consultant rows. All required Excel ranges (months, names, PLAN rows, ACTUAL rows, total row) are automatically generated from this data. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation comment | None | None | Batch retrieval of all ranges, followed by conversion into structured JSON objects for each person, including planned and actual values for each month. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation comment | None | None | At the same time: The uploaded time-tracking CSV file is imported, hours per person per month are totaled, and converted to person-days (PD = hours/8).\n⚠️ Expect a comma as the decimal separator (e.g., “1,00”) and the date format DD.MM.YYYY. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation comment | None | None | Combines all three sources (Template-PLAN, Template-ACTUAL, CSV-ACTUAL) and matches names (full name, last name only, first name only, and also “Name1 / Name2”).\n\nCalculates the deviation between PLAN and IST, including traffic-light logic (🟢 <5% / 🟡 <15% / 🔴 ≥15%), only for the currently reported month. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Documentation comment | None | None | Returns each calculated cell individually via the Graph API (batch loop).\nWrites the formula only if it is missing from the template or is incorrect. |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Documentation comment | None | None | Reads the separate sheet “ Cumulative Basis (Kumulgrundlage)”, checks whether the current month already exists, and whether the cumulative values (previous year's total + new month) are correct.\nUses an Excel workbook session to write the month headers and cumulative values atomically (create a new column OR correct an existing one). |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Documentation comment | None | None | Generates the text summary (monthly & cumulative traffic light), re-downloads the updated template from OneDrive, collects all errors from previous nodes (those running with `continueErrorOutput`), and attaches them to the reply email. Sends it as a reply to the original sender and marks the original email as read. |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Triggers workflow on a schedule | None | Get many messages | |
| Get many messages | `n8n-nodes-base.microsoftOutlook` | Fetches unread emails with attachments | Schedule Trigger | Identify Attachments | |
| Identify Attachments | `n8n-nodes-base.code` | Identifies template and CSV attachments | Get many messages | Upload the template to OneDrive, Extract Acuaul Values "IST" | |
| Upload the template to OneDrive | `n8n-nodes-base.httpRequest` | Uploads Excel template to OneDrive | Identify Attachments | Find the last line | |
| Find the last line | `n8n-nodes-base.httpRequest` | Gets used range of source worksheet | Upload the template to OneDrive | Generate Ranges Dynamically | |
| Generate Ranges Dynamically | `n8n-nodes-base.code` | Generates cell range parameters | Find the last line | Retrieve batch ranges | |
| Retrieve batch ranges  | `n8n-nodes-base.httpRequest` | Fetches range data from Graph API | Generate Ranges Dynamically | structure ranges | |
| structure ranges | `n8n-nodes-base.code` | Structures raw range data | Retrieve batch ranges  | Structure Planned "PLAN"  and Actual "IST" | |
| Structure Planned "PLAN"  and Actual "IST" | `n8n-nodes-base.code` | Parses PLAN and IST structures | structure ranges | Extract PLAN Values, Extract Actual "IST" Values | |
| Extract PLAN Values | `n8n-nodes-base.code` | Filters PLAN dataset | Structure Planned "PLAN"  and Actual "IST" | Merge Data | |
| Extract Actual "IST" Values  | `n8n-nodes-base.code` | Filters IST dataset | Structure Planned "PLAN"  and Actual "IST" | Merge Data | |
| Merge Data | `n8n-nodes-base.merge` | Merges PLAN, IST, and CSV datasets | Aggregate IST (hours → PD), Extract PLAN Values, Extract Actual "IST" Values | Calculate deviations | |
| Calculate deviations | `n8n-nodes-base.code` | Calculates deviations and traffic lights | Merge Data | Enter Actual "IST" Values in the Template | |
| Extract Acuaul Values "IST" | `n8n-nodes-base.extractFromFile` | Parses timesheet CSV file | Identify Attachments | Aggregate IST (hours → PD) | |
| Aggregate IST (hours → PD) | `n8n-nodes-base.code` | Aggregates hours and converts to PD | Extract Acuaul Values "IST" | Merge Data | |
| read Kumulgrundlage  | `n8n-nodes-base.httpRequest` | Reads cumulative base worksheet | Enter Actual "IST" Values in the Template | Calculate cumulative values | |
| Calculate cumulative values | `n8n-nodes-base.code` | Computes cumulative totals | read Kumulgrundlage  | Update Kumulgrundlage? | |
| Update Kumulgrundlage? | `n8n-nodes-base.if` | Evaluates cumulative update condition | Calculate cumulative values | Add a new month, Create  summary | |
| Add a new month | `n8n-nodes-base.httpRequest` | Creates an Excel session | Update Kumulgrundlage? | Find  column | |
| Find  column | `n8n-nodes-base.code` | Determines target column index | Add a new month | Write the month and values | |
| Write the month and values | `n8n-nodes-base.if` | Checks if month exists | Find  column | Set the new month, Update Cumulative Values | |
| Set the new month | `n8n-nodes-base.httpRequest` | Writes new month data | Write the month and values | Merge Values and Close Session | |
| Update Cumulative Values | `n8n-nodes-base.httpRequest` | Updates existing cumulative values | Write the month and values | Merge Values and Close Session | |
| Merge Values and Close Session | `n8n-nodes-base.httpRequest` | Closes active Excel session | Set the new month, Update Cumulative Values | Create  summary | |
| Enter Actual "IST" Values in the Template | `n8n-nodes-base.splitInBatches` | Loops through cell update items | Calculate deviations | read Kumulgrundlage , Check and Write the Formula | |
| Check and Write the Formula | `n8n-nodes-base.if` | Checks formula requirements | Enter Actual "IST" Values in the Template | Update a Single Cell, Enter Actual "IST" Values in the Template | |
| Update a Single Cell | `n8n-nodes-base.httpRequest` | Patches cell values or formulas | Check and Write the Formula | Enter Actual "IST" Values in the Template | |
| Create  summary | `n8n-nodes-base.code` | Generates execution text summary | Update Kumulgrundlage?, Merge Values and Close Session | Download the template from OneDrive | |
| Download the template from OneDrive | `n8n-nodes-base.httpRequest` | Downloads updated workbook binary | Create  summary | Collect errors | |
| Collect errors | `n8n-nodes-base.code` | Compiles error report and summary | Download the template from OneDrive | return Template  | |
| return Template  | `n8n-nodes-base.microsoftOutlook` | Sends email reply with attachment | Collect errors | Mark the email as read | |
| Mark the email as read | `n8n-nodes-base.microsoftOutlook` | Marks incoming message as read | return Template  | None | |
| Sticky Note - Overview | `n8n-nodes-base.stickyNote` | Workflow overview documentation | None | None | ## Update a project controlling Excel template from timesheet CSV emails with Outlook and OneDrive...\n(See full content in JSON source) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Trigger:**
   - Add a `Schedule Trigger` node. Set the interval to every `1` minute.
2. **Configure Outlook Message Retrieval:**
   - Add a `Microsoft Outlook` node configured to **Get Many** messages. Set filters to unread messages with attachments (`readStatus: unread`, `hasAttachments: true`, limit `1`). Enable `downloadAttachments`. Set error handling to continue on error. Connect credentials (`Microsoft Outlook OAuth2 API`).
3. **Add Attachment Parsing Logic:**
   - Add a `Code` node named `Identify Attachments`. Insert JavaScript to iterate over binary files, identifying the Excel template (`projektcontrolling`, `template`, `mto`) and timesheet CSV (`elts`, `zeiterfassung`, `cn`, `.csv`). Throw an error if files are missing.
4. **Upload Template to OneDrive:**
   - Add an `HTTP Request` node (`PUT`) named `Upload the template to OneDrive`. Set the URL to `https://graph.microsoft.com/v1.0/me/drive/root:/ProjectControlling_{{ $execution.id }}_{{ $("Get many messages").first().json.id }}.xlsx:/content`. Set content-type header to `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` and input data field to the template binary key. Connect Microsoft OneDrive OAuth2 credentials.
5. **Inspect Worksheet Ranges:**
   - Add an `HTTP Request` node (`GET`) named `Find the last line` targeting `https://graph.microsoft.com/v1.0/me/drive/items/{{ $("Upload the template to OneDrive").first().json.id }}/workbook/worksheets/Ressourcenverbrauch/usedRange` with header `Prefer: IdType="Id"`.
   - Add a `Code` node named `Generate Ranges Dynamically` to calculate consultant row bounds and define range objects.
   - Add an `HTTP Request` node (`GET`) named `Retrieve batch ranges  ` to fetch range contents using dynamic URLs.
   - Add a `Code` node named `structure ranges` to parse array outputs into structured schema objects.
   - Add a `Code` node named `Structure Planned "PLAN"  and Actual "IST"` to segregate plan and actual items.
   - Add two `Code` nodes named `Extract PLAN Values` and `Extract Actual "IST" Values ` to filter respective items.
6. **Process CSV Timesheet Data:**
   - Add an `Extract From File` node named `Extract Acuaul Values "IST"` configured with a semicolon delimiter (`;`) pointing to the CSV binary property.
   - Add a `Code` node named `Aggregate IST (hours → PD)` to parse dates, normalize commas to decimal points, aggregate hours per person/month, and convert them to person-days (Hours ÷ 8).
7. **Merge Datasets & Calculate Deviations:**
   - Add a `Merge` node configured for 3 inputs, connecting `Aggregate IST (hours → PD)`, `Extract PLAN Values`, and `Extract Actual "IST" Values `.
   - Add a `Code` node named `Calculate deviations` to execute fuzzy name matching, compute variances, assign traffic-light statuses ($\text{Green} < 5\%$, $\text{Yellow} < 15\%$, $\text{Red} \ge 15\%$), and generate cell updates and sum formulas.
8. **Update Workbook Cells (Ressourcenverbrauch):**
   - Add a `Split In Batches` node named `Enter Actual "IST" Values in the Template` to loop through calculated items.
   - Add an `If` node named `Check and Write the Formula` evaluating `IstFormel` and `FormelFehlt`.
   - Add an `HTTP Request` node (`PATCH`) named `Update a Single Cell` targeting cell addresses with values or formulas via Microsoft Graph. Loop output back to the batch node.
9. **Process Cumulative Worksheet (Kumulgrundlage):**
   - Add an `HTTP Request` node (`GET`) named `read Kumulgrundlage ` fetching the used range of the `Kumulgrundlage` sheet.
   - Add a `Code` node named `Calculate cumulative values` to evaluate monthly cumulative totals and determine if updates are required.
   - Add an `If` node named `Update Kumulgrundlage?` checking `aktualisiereKumul`.
   - If true: Add an `HTTP Request` (`POST`) named `Add a new month` to create an Excel session (`/createSession`), followed by a `Code` node (`Find  column`) to locate target columns, an `If` node (`Write the month and values`), and `PATCH` requests (`Set the new month` / `Update Cumulative Values`) using session headers.
   - Terminate the session with an `HTTP Request` (`POST`) named `Merge Values and Close Session` (`/closeSession`).
10. **Compile Summary, Download, and Reply:**
    - Add a `Code` node named `Create  summary` to format execution summaries and traffic light metrics.
    - Add an `HTTP Request` (`GET`) named `Download the template from OneDrive` to fetch the updated binary file (`updated_template`).
    - Add a `Code` node named `Collect errors` to aggregate error outputs across previous nodes.
    - Add a `Microsoft Outlook` node named `return Template ` configured to **Send** a reply email with the summary/error body and attached binary file to the original sender.
    - Add a `Microsoft Outlook` node named `Mark the email as read` configured to **Update** the message read status (`isRead: true`). Connect credentials (`Microsoft Outlook OAuth2 API`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Project Controlling Assistant Documentation & Overview | Provided via inline workflow sticky notes and configuration parameters. |
| Microsoft Graph REST API Reference | [Microsoft Graph Excel API Documentation](https://learn.microsoft.com/en-us/graph/api/resources/excel) |