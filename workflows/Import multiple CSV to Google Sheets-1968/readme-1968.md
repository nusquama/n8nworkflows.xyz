Import multiple CSV to Google Sheets

https://n8nworkflows.xyz/workflows/import-multiple-csv-to-google-sheets-1968


# Import multiple CSV to Google Sheets

### 1. Workflow Overview

This workflow automates the importation and consolidation of multiple CSV files into a single Google Sheets document. It is designed to process CSV files found in a specified directory, parse their contents, clean and filter the data, sort it, and then append or update the records within a target Google Sheet. The use cases include subscriber list management, data aggregation from multiple sources, and ongoing data synchronization tasks where duplicates are removed, and only relevant, subscribed users are maintained.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Input Reception:** Initiates the workflow manually and reads CSV files from a directory.
- **1.2 CSV Data Parsing & Annotation:** Parses individual CSV files and tags data with its source filename.
- **1.3 Data Cleaning & Filtering:** Removes duplicates, filters for subscribed users, and sorts data by subscription date.
- **1.4 Data Upload:** Appends or updates the cleaned data into a specified Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

**Overview:**  
This block starts the workflow on manual trigger and loads all CSV files from a predefined directory as binary data.

**Nodes Involved:**  
- When clicking "Execute Workflow"  
- Read Binary Files  
- Split In Batches

**Node Details:**

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Initiates workflow execution on user command.  
  - Configuration: Default manual trigger settings, no parameters.  
  - Inputs: None  
  - Outputs: Connected to "Read Binary Files"  
  - Failure cases: None expected unless manual execution is interrupted.

- **Read Binary Files**  
  - Type: Read Binary Files  
  - Role: Reads all CSV files from a local directory.  
  - Configuration: File selector set to "./.n8n/*.csv" — reads all CSV files in the .n8n folder.  
  - Inputs: Trigger node output  
  - Outputs: Binary data of each CSV file to "Split In Batches"  
  - Failure cases: File access permission errors, missing directory, empty directory (no files).  
  - Notes: If no files found, subsequent nodes may not execute.

- **Split In Batches**  
  - Type: Split In Batches  
  - Role: Processes files one at a time by splitting the binary data into batches of size 1.  
  - Configuration: Batch size = 1  
  - Inputs: Binary files from "Read Binary Files"  
  - Outputs: Two outputs: first goes to "Read CSV" for parsing, second goes to "Remove duplicates" (used after annotation)  
  - Failure cases: Batch processing errors, misconfiguration of batch size.

#### 1.2 CSV Data Parsing & Annotation

**Overview:**  
Reads each CSV file content into JSON format and annotates each dataset with its source filename for traceability.

**Nodes Involved:**  
- Read CSV  
- Assign source file name

**Node Details:**

- **Read CSV**  
  - Type: Spreadsheet File (CSV reader)  
  - Role: Parses CSV binary data into structured JSON.  
  - Configuration:  
    - rawData: true (keep original data types)  
    - headerRow: true (first row as header)  
    - readAsString: true (read all data as string)  
    - includeEmptyCells: false (omit empty cells)  
    - fileFormat: CSV  
  - Inputs: Binary data from "Split In Batches"  
  - Outputs: Parsed JSON data to "Assign source file name"  
  - Failure cases: Malformed CSV files, encoding issues, missing headers.

- **Assign source file name**  
  - Type: Set  
  - Role: Adds a new field "Source" to each data item containing the file name it originated from.  
  - Configuration: Sets a string field "Source" using expression `={{ $('Split In Batches').item.binary.data.fileName }}` to dynamically capture the file name.  
  - Inputs: Parsed JSON data from "Read CSV"  
  - Outputs: Annotated data back to "Split In Batches" for further processing  
  - Failure cases: Expression errors if upstream node outputs missing or inconsistent.

#### 1.3 Data Cleaning & Filtering

**Overview:**  
Removes duplicate user entries based on username, filters for only subscribed users, and sorts the data chronologically by subscription date.

**Nodes Involved:**  
- Remove duplicates  
- Keep only subscribers  
- Sort by date

**Node Details:**

- **Remove duplicates**  
  - Type: Item Lists (Remove Duplicates)  
  - Role: Removes duplicate entries based on the 'user_name' field.  
  - Configuration:  
    - Operation: removeDuplicates  
    - Compare: selectedFields  
    - Fields to compare: "user_name"  
  - Inputs: Annotated JSON data from "Split In Batches" (output of "Assign source file name")  
  - Outputs: Unique user data to "Keep only subscribers"  
  - Failure cases: Missing 'user_name' field, case sensitivity issues.

- **Keep only subscribers**  
  - Type: Filter  
  - Role: Filters data to retain only entries where 'subscribed' field equals "TRUE".  
  - Configuration: Condition on string equality `{ $json.subscribed === "TRUE" }`  
  - Inputs: Unique user data from "Remove duplicates"  
  - Outputs: Filtered data to "Sort by date"  
  - Failure cases: Field missing or misformatted 'subscribed' values.

- **Sort by date**  
  - Type: Item Lists (Sort)  
  - Role: Sorts the filtered data by the 'date_subscribed' field in ascending order.  
  - Configuration:  
    - Operation: sort  
    - Sort field: "date_subscribed"  
  - Inputs: Filtered subscriber data from "Keep only subscribers"  
  - Outputs: Sorted data to "Upload to spreadsheet"  
  - Failure cases: Non-uniform date formats, missing dates.

#### 1.4 Data Upload

**Overview:**  
Uploads the cleaned, filtered, and sorted data into a Google Sheets document by appending new records or updating existing ones based on the username.

**Nodes Involved:**  
- Upload to spreadsheet

**Node Details:**

- **Upload to spreadsheet**  
  - Type: Google Sheets  
  - Role: Appends or updates rows in a Google Sheet based on the 'user_name' column.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Document ID: URL pointing to the target Google Sheets document (`https://docs.google.com/spreadsheets/d/13YYuEJ1cDf-t8P2MSTFWnnNHCreQ6Zo8oPSp7WeNnbY`)  
    - Sheet Name: Selected by sheet ID (2042396108)  
    - Columns mapped automatically with matching columns set on "user_name" for update logic  
  - Inputs: Sorted subscriber data from "Sort by date"  
  - Outputs: None (end of workflow)  
  - Credentials: Requires Google Sheets OAuth2 credentials configured as `Google Sheets account`  
  - Failure cases: Authentication errors, API rate limits, incorrect sheet or document ID, permission issues, data mapping errors.

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                    | Input Node(s)               | Output Node(s)              | Sticky Note                               |
|-----------------------------|-----------------------|----------------------------------|----------------------------|-----------------------------|-------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger        | Starts workflow execution        | -                          | Read Binary Files            |                                           |
| Read Binary Files           | Read Binary Files      | Reads CSV files from directory   | When clicking "Execute Workflow" | Split In Batches             |                                           |
| Split In Batches            | Split In Batches       | Processes one file per batch     | Read Binary Files           | Read CSV, Remove duplicates  |                                           |
| Read CSV                   | Spreadsheet File       | Parses CSV content to JSON       | Split In Batches            | Assign source file name      |                                           |
| Assign source file name     | Set                   | Adds source file name to data    | Read CSV                   | Split In Batches             |                                           |
| Remove duplicates           | Item Lists            | Removes duplicate usernames      | Split In Batches            | Keep only subscribers        |                                           |
| Keep only subscribers       | Filter                | Keeps only subscribed entries    | Remove duplicates           | Sort by date                |                                           |
| Sort by date               | Item Lists            | Sorts data by subscription date | Keep only subscribers       | Upload to spreadsheet        |                                           |
| Upload to spreadsheet       | Google Sheets          | Appends or updates Google Sheet  | Sort by date               | -                           | Requires Google Sheets OAuth2 credentials |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add a Read Binary Files node:**  
   - Connect from Manual Trigger.  
   - Set "File Selector" to `=./.n8n/*.csv` to select all CSV files in the `.n8n` folder.

3. **Add a Split In Batches node:**  
   - Connect from Read Binary Files.  
   - Set "Batch Size" to 1 to process files individually.

4. **Add a Spreadsheet File node:**  
   - Connect from the first output of Split In Batches.  
   - Configure as:  
     - File Format: CSV  
     - Options:  
       - rawData: true  
       - headerRow: true  
       - readAsString: true  
       - includeEmptyCells: false

5. **Add a Set node named "Assign source file name":**  
   - Connect from Spreadsheet File node.  
   - Add a field named "Source" with value expression: `={{ $('Split In Batches').item.binary.data.fileName }}`

6. **Connect the Set node back into the second output of Split In Batches:**  
   - This reconnects processed data for further steps.

7. **Add an Item Lists node configured to Remove Duplicates:**  
   - Connect from Split In Batches (output after the Set node).  
   - Operation: Remove Duplicates  
   - Compare: Selected Fields  
   - Fields to Compare: `user_name`

8. **Add a Filter node named "Keep only subscribers":**  
   - Connect from Remove Duplicates.  
   - Condition: String equals  
     - Field: `{{$json.subscribed}}`  
     - Value: `TRUE`

9. **Add an Item Lists node to Sort by Date:**  
   - Connect from Keep only subscribers.  
   - Operation: Sort  
   - Sort Field: `date_subscribed` (ascending)

10. **Add a Google Sheets node named "Upload to spreadsheet":**  
    - Connect from Sort by date.  
    - Operation: Append Or Update  
    - Document ID: Set using the URL of the target Google Sheet (e.g., `https://docs.google.com/spreadsheets/d/...`)  
    - Sheet Name: Select the target sheet by name or ID.  
    - Columns: Automap input data; set `user_name` as the matching column for updates.  
    - Credentials: Configure OAuth2 credentials for Google Sheets API access.

11. **Save and test the workflow:**  
    - Click "Execute Workflow" manually to start processing.  
    - Monitor for errors and verify data is appended/updated correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow assumes all CSV files are located in the `.n8n` directory relative to the workflow execution context. | File reading path: `=./.n8n/*.csv`                                                                  |
| Google Sheets API credentials must be properly configured with OAuth2 and sufficient permissions to edit sheets. | Credential setup in n8n under Google Sheets OAuth2 API                                              |
| The duplicate removal relies on exact matches in `user_name`; consider case normalization if inconsistent data. | Potential enhancement: add a lowercase transform before duplicate removal                           |
| Date sorting assumes consistent date formats in `date_subscribed`; inconsistent formats could cause sorting issues. | Consider adding date parsing/validation steps if date formats vary                                  |
| The workflow is triggered manually; for automation, consider replacing the trigger with a schedule or webhook.   | n8n supports Cron and Webhook triggers                                                              |
| For large CSV files or directories, batch size and Google Sheets API rate limits should be monitored.             | Google Sheets API quotas: https://developers.google.com/sheets/api/limits                            |
| This workflow does not handle CSV encoding issues explicitly; ensure CSV files are UTF-8 encoded.                 | Non-UTF-8 files may cause parse errors in "Read CSV" node                                          |

---

This documentation provides a detailed understanding of the workflow’s structure, node functions, and configuration, enabling reproduction, modification, and troubleshooting.