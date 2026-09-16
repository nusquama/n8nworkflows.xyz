Evaluate KYB risk scores from Google Sheets with Ambriel and save to Google Drive

https://n8nworkflows.xyz/workflows/evaluate-kyb-risk-scores-from-google-sheets-with-ambriel-and-save-to-google-drive-17626


# Evaluate KYB risk scores from Google Sheets with Ambriel and save to Google Drive

### 1. Workflow Overview

This workflow automates the Know Your Business (KYB) risk evaluation process for companies listed in a Google Sheets document. It operates on a periodic schedule, reads company registration records, interfaces with the Ambriel KYB API to evaluate risk scores and generate assessment reports, updates the source spreadsheet with the findings, and archives the generated report files into a designated Google Drive folder.

The logic is grouped into five functional blocks:
- **1.1 Schedule & Intake:** Triggers the workflow execution on a regular monthly interval and fetches company records from a Google Sheets spreadsheet.
- **1.2 Batch Processing & Evaluation:** Iterates through company records sequentially, submits company registration details to Ambriel for KYB assessment with forced report generation, and feeds the item loop.
- **1.3 Sheet Synchronization:** Writes the returned risk level, score, and associated metadata back to the corresponding row in the source Google Sheets document.
- **1.4 Report Verification:** Requests the report file details from Ambriel using the assessment ID and evaluates whether the report is successfully available via a conditional check.
- **1.5 Archiving & Storage:** Downloads the verified report file via an HTTP GET request and uploads it directly to a specified folder within Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Intake
This block initializes the automation run on a fixed schedule and retrieves the target company dataset from the cloud spreadsheet.

- **Yearly Schedule Trigger**
  - **Type & Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node) - Initiates the workflow pipeline.
  - **Configuration:** Configured with an interval rule set to run every month (`field: "months"`).
  - **Expressions:** None.
  - **Connections:** Output connects to *Read from Google Sheet*.
  - **Version Requirements:** Type version 1.3.
  - **Edge Cases & Failure Types:** Service downtime or missed triggers if the n8n instance is offline during the scheduled interval.

- **Read from Google Sheet**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Action Node) - Reads all company records from the specified spreadsheet.
  - **Configuration:** Uses Google Sheets OAuth2 API credentials. Configured to read from document URL `https://docs.google.com/spreadsheets/d/14U1aL0rX8w9BZbVfFD-w9GwPlpSMPQ1J0odYBr7glyI/edit?gid=0#gid=0` on `Sheet1` (`gid=0`).
  - **Expressions:** None.
  - **Connections:** Input from *Yearly Schedule Trigger*; output connects to *Batch Process Items*.
  - **Version Requirements:** Type version 4.7.
  - **Edge Cases & Failure Types:** Authentication failures, invalid document IDs, missing columns (`row_number`, `regcom`), or API rate limits.

---

#### 2.2 Batch Processing & Evaluation
This block handles iterative queue management and submits company data to the external KYB provider.

- **Batch Process Items**
  - **Type & Role:** `n8n-nodes-base.splitInBatches` (Flow Control Node) - Splits incoming list items into manageable batches (default batch size of 1 item) to process records sequentially.
  - **Configuration:** Standard options.
  - **Expressions:** None.
  - **Connections:** Input from *Read from Google Sheet*; output loop 0 connects to *Evaluate Company with Ambriel*, and loop finish (index 0) terminates.
  - **Version Requirements:** Type version 3.
  - **Edge Cases & Failure Types:** Infinite loops if loop resets are misconfigured; memory overhead if batch sizes are set too high for large datasets.

- **Evaluate Company with Ambriel**
  - **Type & Role:** `CUSTOM.ambrielCompany` (Action Node) - Performs the KYB company assessment via the Ambriel API.
  - **Configuration:** Uses Ambriel API credentials. Operation set to `assess`, output mode set to `raw`, country code set to `RO` (Romania), and `forceReportGenerate` enabled (`true`).
  - **Expressions:** `registrationNumber`: `={{ $json.regcom }}`
  - **Connections:** Input from *Batch Process Items*; output splits into three parallel branches connecting to *Batch Process Items* (to continue loop), *Update Google Sheet Row*, and *Fetch Ambriel Report File*.
  - **Version Requirements:** Type version 1.
  - **Edge Cases & Failure Types:** API timeouts, invalid registration numbers (`regcom`), invalid country codes, or invalid Ambriel API credentials.

---

#### 2.3 Sheet Synchronization
This block records the results of the automated assessment back into the source document.

- **Update Google Sheet Row**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Action Node) - Updates an existing row in the spreadsheet with evaluation results.
  - **Configuration:** Uses Google Sheets OAuth2 API credentials. Operation set to `update`, mapping mode set to define below, matching columns set to `row_number`, and cell format set to `USER_ENTERED`.
  - **Expressions:** 
    - `cui`: `={{ $('Batch Process Items').item.json.cui }}`
    - `name`: `={{ $('Batch Process Items').item.json.name }}`
    - `risk`: `={{ $json.level }}`
    - `score`: `={{ $json.score }}`
    - `regcom`: `={{ $('Batch Process Items').item.json.regcom }}`
    - `row_number`: `={{ $('Batch Process Items').item.json.row_number }}`
  - **Connections:** Input from *Evaluate Company with Ambriel*; no downstream outputs.
  - **Version Requirements:** Type version 4.7.
  - **Edge Cases & Failure Types:** Matching failure if `row_number` is missing or mismatched; column schema mismatches.

---

#### 2.4 Report Verification
This block checks whether a downloadable assessment report was successfully generated by the external API.

- **Fetch Ambriel Report File**
  - **Type & Role:** `CUSTOM.ambrielCompany` (Action Node) - Requests metadata and download links for the generated assessment report.
  - **Configuration:** Uses Ambriel API credentials. Operation set to `getReport`.
  - **Expressions:** `assessmentId`: `={{ $json.id }}`
  - **Connections:** Input from *Evaluate Company with Ambriel*; output connects to *If Report Success*.
  - **Version Requirements:** Type version 1.
  - **Edge Cases & Failure Types:** Assessment ID not found or report generation still in progress (returning a failure status).

- **If Report Success**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control Node) - Evaluates whether the report fetch operation succeeded.
  - **Configuration:** Condition combinator set to `and` with strict type validation.
  - **Expressions:** Left value: `={{ $json.success }}`, Operator: Boolean Equals, Right value: `true`.
  - **Connections:** Input from *Fetch Ambriel Report File*; true branch output connects to *Download Report File*.
  - **Version Requirements:** Type version 2.3.
  - **Edge Cases & Failure Types:** Schema changes in the Ambriel API response causing boolean evaluation mismatches.

---

#### 2.5 Archiving & Storage
This block retrieves the physical report file over HTTP and stores it in cloud storage.

- **Download Report File**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (Action Node) - Downloads the binary report file from the URL provided by Ambriel.
  - **Configuration:** Standard HTTP GET request. Retry on fail disabled, always output data disabled.
  - **Expressions:** `url`: `={{ $('Fetch Ambriel Report File').item.json.url }}`
  - **Connections:** Input from *If Report Success*; output connects to *Upload to Google Drive*.
  - **Version Requirements:** Type version 4.4.
  - **Edge Cases & Failure Types:** Expired download URLs, network connectivity issues, or HTTP 4xx/5xx errors from the hosting provider.

- **Upload to Google Drive**
  - **Type & Role:** `n8n-nodes-base.googleDrive` (Action Node) - Uploads the downloaded binary report file to a specific Google Drive folder.
  - **Configuration:** Uses Google Drive OAuth2 API credentials. Drive ID set to `My Drive`, destination folder ID set to `114Zo2QpcnXnxylAqMa8yl3NbEpkVy5Uh` (Ambriel KYB).
  - **Expressions:** `name`: `={{ $binary.data.fileName }}`
  - **Connections:** Input from *Download Report File*; no downstream outputs.
  - **Version Requirements:** Type version 3.
  - **Edge Cases & Failure Types:** Insufficient permissions on the destination folder, storage quota exhaustion, or missing binary data.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Yearly Schedule Trigger | n8n-nodes-base.scheduleTrigger | Initiates workflow monthly | None | Read from Google Sheet | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process. |
| Read from Google Sheet | n8n-nodes-base.googleSheets | Reads company records from sheet | Yearly Schedule Trigger | Batch Process Items | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process.<br><br>## Schedule sheet intake<br><br>Starts the workflow on a schedule and reads the rows to be evaluated from Google Sheets. |
| Batch Process Items | n8n-nodes-base.splitInBatches | Splits rows into sequential batches | Read from Google Sheet, Evaluate Company with Ambriel | Evaluate Company with Ambriel | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process.<br><br>## Evaluate and update rows<br><br>Loops through each sheet row, runs the Ambriel company KYB evaluation, feeds the loop for the next item, and writes evaluation results back to Google Sheets. |
| Evaluate Company with Ambriel | CUSTOM.ambrielCompany | Assesses company KYB via API | Batch Process Items | Batch Process Items, Update Google Sheet Row, Fetch Ambriel Report File | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process.<br><br>## Evaluate and update rows<br><br>Loops through each sheet row, runs the Ambriel company KYB evaluation, feeds the loop for the next item, and writes evaluation results back to Google Sheets. |
| Update Google Sheet Row | n8n-nodes-base.googleSheets | Writes score and risk back to sheet | Evaluate Company with Ambriel | None | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process.<br><br>## Evaluate and update rows<br><br>Loops through each sheet row, runs the Ambriel company KYB evaluation, feeds the loop for the next item, and writes evaluation results back to Google Sheets. |
| Fetch Ambriel Report File | CUSTOM.ambrielCompany | Requests report details from Ambriel | Evaluate Company with Ambriel | If Report Success | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process.<br><br>## Check report availability<br><br>Requests the Ambriel report file details for the evaluated company and uses a conditional branch to continue only when a report file is available. |
| If Report Success | n8n-nodes-base.if | Conditional check for report availability | Fetch Ambriel Report File | Download Report File | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process.<br><br>## Check report availability<br><br>Requests the Ambriel report file details for the evaluated company and uses a conditional branch to continue only when a report file is available. |
| Download Report File | n8n-nodes-base.httpRequest | Downloads report file binary via HTTP | If Report Success | Upload to Google Drive | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process.<br><br>## Download and store report<br><br>Downloads the available report file by HTTP request and uploads the resulting file to Google Drive. |
| Upload to Google Drive | n8n-nodes-base.googleDrive | Uploads report binary to Google Drive | Download Report File | None | ## Google Sheets KYB Evaluation + Reports on Drive<br><br>### How it works<br><br>This workflow runs on a schedule, reads company records from Google Sheets, and processes each row through Ambriel for KYB evaluation. It updates the source sheet with evaluation results, checks whether a report file is available, then downloads and uploads the report to Google Drive.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger with the desired run frequency.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, and row range containing the companies to evaluate.<br>- Configure Ambriel credentials and map the required company fields from the sheet into the evaluation and report-file nodes.<br>- Connect Google Drive credentials and choose the destination folder for uploaded reports.<br>- Verify the If condition matches the expected Ambriel response field that indicates a report file is available.<br><br>### Customization<br><br>Adjust the Google Sheets column mappings, Ambriel evaluation inputs, report availability condition, and Google Drive destination folder to match your KYB process.<br><br>## Download and store report<br><br>Downloads the available report file by HTTP request and uploads the resulting file to Google Drive. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps sequentially to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`Yearly Schedule Trigger`).
   - Set the interval rule to run every month (`field: "months"`).

2. **Add the Google Sheets Read Node:**
   - Add a **Google Sheets** node (`Read from Google Sheet`).
   - Connect it to the output of the *Yearly Schedule Trigger*.
   - Configure credentials using a Google Sheets OAuth2 account.
   - Set the Document ID using the target spreadsheet URL and select `Sheet1` (`gid=0`).

3. **Add the Batch Processing Node:**
   - Add a **Split In Batches** node (`Batch Process Items`).
   - Connect it to the output of *Read from Google Sheet*.
   - Keep default batch settings.

4. **Add the Ambriel Assessment Node:**
   - Add an **Ambriel** custom node (`Evaluate Company with Ambriel`).
   - Connect loop output 0 of *Batch Process Items* to this node.
   - Configure credentials using an Ambriel API account.
   - Set operation to `assess`, output mode to `raw`, country code to `RO`, and enable `forceReportGenerate` (`true`).
   - Set the registration number parameter using the expression: `={{ $json.regcom }}`.

5. **Connect the Loop Feedback & Sheet Update Nodes:**
   - Connect the output of *Evaluate Company with Ambriel* back to *Batch Process Items* to advance the loop.
   - Add a **Google Sheets** node (`Update Google Sheet Row`).
   - Connect the output of *Evaluate Company with Ambriel* to this node.
   - Configure Google Sheets OAuth2 credentials, select the same spreadsheet/sheet, set operation to `update`, matching column to `row_number`, and cell format to `USER_ENTERED`.
   - Map columns using expressions:
     - `cui`: `={{ $('Batch Process Items').item.json.cui }}`
     - `name`: `={{ $('Batch Process Items').item.json.name }}`
     - `risk`: `={{ $json.level }}`
     - `score`: `={{ $json.score }}`
     - `regcom`: `={{ $('Batch Process Items').item.json.regcom }}`
     - `row_number`: `={{ $('Batch Process Items').item.json.row_number }}`

6. **Add the Ambriel Report Retrieval Node:**
   - Add an **Ambriel** custom node (`Fetch Ambriel Report File`).
   - Connect the output of *Evaluate Company with Ambriel* to this node.
   - Configure Ambriel API credentials and set operation to `getReport`.
   - Set the assessment ID parameter using the expression: `={{ $json.id }}`.

7. **Add the Conditional Check Node:**
   - Add an **If** node (`If Report Success`).
   - Connect the output of *Fetch Ambriel Report File* to this node.
   - Configure a condition where the left value `={{ $json.success }}` equals boolean `true`.

8. **Add the HTTP Download Node:**
   - Add an **HTTP Request** node (`Download Report File`).
   - Connect the true branch output of *If Report Success* to this node.
   - Set method to GET and set the URL using the expression: `={{ $('Fetch Ambriel Report File').item.json.url }}`.

9. **Add the Google Drive Upload Node:**
   - Add a **Google Drive** node (`Upload to Google Drive`).
   - Connect the output of *Download Report File* to this node.
   - Configure credentials using a Google Drive OAuth2 account.
   - Set Drive ID to `My Drive`, select the destination folder ID (`114Zo2QpcnXnxylAqMa8yl3NbEpkVy5Uh`), and set the file name using the expression: `={{ $binary.data.fileName }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets Template Source | [Google Sheets Spreadsheet](https://docs.google.com/spreadsheets/d/14U1aL0rX8w9BZbVfFD-w9GwPlpSMPQ1J0odYBr7glyI/edit?gid=0#gid=0) |
| Google Drive Folder Destination | [Google Drive Folder](https://drive.google.com/drive/folders/114Zo2QpcnXnxylAqMa8yl3NbEpkVy5Uh) |