Automate Payment Gateway & Database Transaction Reconciliation with Google Sheets

https://n8nworkflows.xyz/workflows/automate-payment-gateway---database-transaction-reconciliation-with-google-sheets-11684


# Automate Payment Gateway & Database Transaction Reconciliation with Google Sheets

### 1. Workflow Overview

This workflow automates the reconciliation process between transaction records from a Local Database and a Payment Gateway by leveraging Google Sheets as data sources and sinks. It is designed for financial teams or automation engineers who need to ensure data consistency and integrity between internal records and external payment systems.

The workflow is structured into the following logical blocks:

- **1.1 Input Data Retrieval:** Reads transaction data from Google Sheets representing the Local Database and Payment Gateway.
- **1.2 Transaction Comparison:** Compares datasets to identify valid matches, invalid entries, missing transactions, and amount discrepancies.
- **1.3 Duplicate Detection:** Scans for duplicate transactions in the Local Database dataset.
- **1.4 Logging Discrepancies:** Writes categorized discrepancies into dedicated Google Sheets tabs for audit and review.
- **1.5 Summary Aggregation:** Counts each category of transactions and merges these counts into a unified summary.
- **1.6 Reporting:** Writes the final summary to a reconciliation Google Sheet and sends an email report to the finance team.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Data Retrieval

**Overview:**  
Fetches transaction data from two Google Sheets: one representing the Local Database transactions and the other representing Payment Gateway transactions. These datasets serve as the foundation for subsequent comparison and reconciliation.

**Nodes Involved:**  
- `Transactions from Local Database`  
- `Transaction from Payment Gateway`

**Node Details:**  

- **Transactions from Local Database**  
  - Type: Google Sheets  
  - Role: Reads transaction data from the Local Database sheet.  
  - Configuration: Reads from a predefined Google Sheet and specific tab containing columns like `PaymentRef` and `Amount`. Uses a Google Service Account for authentication.  
  - Inputs: Triggered by the schedule trigger.  
  - Outputs: Data passed to comparison and duplicate detection nodes.  
  - Failure Modes: Authentication errors with Google API; missing or malformed data in the sheet; network timeouts.

- **Transaction from Payment Gateway**  
  - Type: Google Sheets  
  - Role: Reads transaction data from the Payment Gateway sheet.  
  - Configuration: Similar setup as above, reading from a separate Google Sheet tab with `PaymentRef` and `Amount`. Authenticated via Google Service Account.  
  - Inputs: Triggered by the schedule trigger.  
  - Outputs: Data sent to the comparison node.  
  - Failure Modes: Same as above.

#### 2.2 Transaction Comparison

**Overview:**  
Compares transactions from the Local Database and Payment Gateway datasets based on the `PaymentRef` field to categorize transactions into valid, invalid, missing, and amount difference groups.

**Nodes Involved:**  
- `Compare both transactions`  
- `Valid Transactions`  
- `Invalid transaction`  
- `Add missing transaction to Sheet`  
- `Finding Amount Difference`  
- `Adding Amount difference transactions`

**Node Details:**  

- **Compare both transactions**  
  - Type: Compare Datasets (Merge node in compare mode)  
  - Role: Performs a field-based comparison between Local Database and Payment Gateway transactions using `PaymentRef` as the key.  
  - Configuration: Merge configured to compare on `PaymentRef`. Outputs split into: matched (valid), differing, only in A (Local Database), and only in B (Payment Gateway).  
  - Inputs: Receives datasets from both Google Sheets nodes.  
  - Outputs: Four outputs feeding nodes that log valid, invalid, missing, and differing transactions.  
  - Edge Cases: Missing keys, inconsistent field names, or data types can cause mismatches or errors.

- **Valid Transactions**  
  - Type: Google Sheets (Append/Update)  
  - Role: Logs transactions that match on `PaymentRef` and have consistent data.  
  - Configuration: Writes to a Google Sheet tab labeled `RealData`. Maps `PaymentRef` and `Amount`.  
  - Inputs: Receives valid transactions from compare node.  
  - Outputs: Feeds counting nodes.  
  - Failures: Google API errors, schema mismatches.

- **Invalid transaction**  
  - Type: Google Sheets (Append/Update)  
  - Role: Logs transactions that mismatch or are invalid.  
  - Configuration: Writes `PaymentRef` to a sheet named `DataNotInsert`.  
  - Inputs: Receives invalid transactions from compare node.  
  - Outputs: Feeds counting nodes.

- **Add missing transaction to Sheet**  
  - Type: Google Sheets (Append/Update)  
  - Role: Logs transactions missing in one dataset.  
  - Configuration: Writes `PaymentRef` to `DataNotInsert` sheet (same as invalid transactions).  
  - Inputs: Receives missing transactions from compare node.  
  - Outputs: Feeds counting nodes.

- **Finding Amount Difference**  
  - Type: Filter  
  - Role: Filters transactions where the `Amount` differs between Local Database and Payment Gateway.  
  - Configuration: Condition checks if `Amount` in Local DB ≠ `Amount` in Payment Gateway.  
  - Inputs: Receives differing transactions from compare node.  
  - Outputs: Sends filtered transactions to add to discrepancy sheet.

- **Adding Amount difference transactions**  
  - Type: Google Sheets (Append/Update)  
  - Role: Logs transactions with amount differences.  
  - Configuration: Writes `PaymentRef`, `DBAmount`, and `PortalAmount` to `AmountDifference` sheet.  
  - Inputs: Filtered amount difference transactions.  
  - Outputs: Feeds counting nodes.

#### 2.3 Duplicate Detection

**Overview:**  
Detects duplicate transactions in the Local Database dataset based on repeated `PaymentRef` values and logs them for audit.

**Nodes Involved:**  
- `Finding duplicate transactions` (disabled)  
- `Adding duplicate transactions`

**Node Details:**  

- **Finding duplicate transactions**  
  - Type: Code (JavaScript)  
  - Role: Iterates over the Local Database transactions to find duplicates by `PaymentRef`.  
  - Configuration: Checks for repeated `PaymentRef` and collects duplicates.  
  - Inputs: Local Database transactions.  
  - Outputs: Duplicate transactions to Google Sheets logging node.  
  - Note: This node is currently disabled, indicating duplicates are not processed or logged in the current workflow run.

- **Adding duplicate transactions**  
  - Type: Google Sheets (Append/Update)  
  - Role: Logs duplicates found into a dedicated `DuplicateData` sheet.  
  - Configuration: Auto-maps input data.  
  - Inputs: Receives duplicates from the code node (would be none while disabled).  
  - Outputs: None.

#### 2.4 Logging Discrepancies and Counts

**Overview:**  
Logs all categories of transactions (valid, invalid, missing, amount differences) into respective Google Sheets tabs and counts the number of each category.

**Nodes Involved:**  
- `Count valid transactions`  
- `Count invalid transactions`  
- `Count Number of Missing Transactions`  
- `Count Number of Amount Difference Transactions.`  
- `Merge with all transactions count`  
- `Code in JavaScript`

**Node Details:**  

- **Count valid transactions**  
  - Type: Summarize  
  - Role: Counts unique `PaymentRef` from valid transactions.  
  - Inputs: From `Valid Transactions` node.  
  - Outputs: To merge counts node.

- **Count invalid transactions**  
  - Type: Summarize  
  - Role: Counts unique `PaymentRef` from invalid transactions.  
  - Inputs: From `Invalid transaction` node.  
  - Outputs: To merge counts node.

- **Count Number of Missing Transactions**  
  - Type: Summarize  
  - Role: Counts unique `PaymentRef` and possibly `MissingTransaction` fields from missing transactions.  
  - Inputs: From `Add missing transaction to Sheet` node.  
  - Outputs: To merge counts node.

- **Count Number of Amount Difference Transactions.**  
  - Type: Summarize  
  - Role: Counts unique `PaymentRef` and `Amount Difference` field occurrences.  
  - Inputs: From `Adding Amount difference transactions` node.  
  - Outputs: To merge counts node.

- **Merge with all transactions count**  
  - Type: Merge (4 inputs)  
  - Role: Combines all four counts into one consolidated dataset.  
  - Inputs: Valid, invalid, missing, and amount difference counts.  
  - Outputs: To JavaScript code node.

- **Code in JavaScript**  
  - Type: Code (JavaScript)  
  - Role: Constructs a summary object aggregating all counts into one JSON object with keys: `ValidTransactions`, `InvalidTransactions`, `AmountDiffereceTransactions`, and `MissingTransactions`.  
  - Inputs: Merged counts node output.  
  - Outputs: To final summary Google Sheet logging node.

#### 2.5 Reporting

**Overview:**  
Writes the final reconciliation counts to a designated Google Sheet tab and sends an email report to the finance team summarizing the day's reconciliation results.

**Nodes Involved:**  
- `Adding the final count of transactions`  
- `Send report to reconciliation team`

**Node Details:**  

- **Adding the final count of transactions**  
  - Type: Google Sheets (Append/Update)  
  - Role: Writes the aggregated summary counts (valid, invalid, missing, amount differences) to the `Reconciliation Summary` sheet.  
  - Configuration: Maps each count field to the sheet columns.  
  - Inputs: From JavaScript summary code node.  
  - Outputs: To email sending node.

- **Send report to reconciliation team**  
  - Type: Email Send  
  - Role: Sends a plain text email to a specified finance team email address containing the reconciliation summary.  
  - Configuration: Uses SMTP credentials; email body uses expressions to dynamically insert counts from the JSON data.  
  - Inputs: From final summary Google Sheets node.  
  - Outputs: None.  
  - Edge Cases: SMTP authentication failures, invalid email addresses, or network issues.

---

### 3. Summary Table

| Node Name                       | Node Type                       | Functional Role                              | Input Node(s)                                   | Output Node(s)                          | Sticky Note                                                                                                        |
|--------------------------------|--------------------------------|----------------------------------------------|------------------------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger               | Starts workflow on schedule                   | -                                              | Transactions from Local Database, Transaction from Payment Gateway |                                                                                                                     |
| Transactions from Local Database| Google Sheets                 | Reads Local Database transactions             | Schedule Trigger                               | Compare both transactions, Finding duplicate transactions | Transactions from Local and Payment Gateway: Reads internal transactions for reference dataset.                     |
| Transaction from Payment Gateway| Google Sheets                 | Reads Payment Gateway transactions            | Schedule Trigger                               | Compare both transactions              | Transactions from Local and Payment Gateway: Reads payment gateway transactions for comparison.                      |
| Compare both transactions       | Compare Datasets (Merge)       | Compares datasets by PaymentRef                | Transactions from Local Database, Transaction from Payment Gateway | Invalid transaction, Valid Transactions, Finding Amount Difference, Add missing transaction to Sheet | Compare Both Transactions: Compares datasets and splits into categories like valid, invalid, missing, differences.   |
| Finding duplicate transactions  | Code (JavaScript) (disabled)   | Finds duplicate PaymentRef in Local Database  | Transactions from Local Database                | Adding duplicate transactions          | Duplicate Transaction Processing: Identifies and logs duplicate PaymentRef entries (currently disabled).            |
| Adding duplicate transactions   | Google Sheets                 | Logs duplicate transactions                    | Finding duplicate transactions                   | -                                     | Duplicate Transaction Processing: Logs duplicates for audit purposes.                                               |
| Valid Transactions              | Google Sheets                 | Logs valid matched transactions                | Compare both transactions                        | Count valid transactions               | Comparisons of all transactions: Logs valid transactions to RealData sheet.                                         |
| Invalid transaction             | Google Sheets                 | Logs invalid/mismatched transactions           | Compare both transactions                        | Count invalid transactions             | Comparisons of all transactions: Logs invalid transactions to DataNotInsert sheet.                                  |
| Add missing transaction to Sheet| Google Sheets                | Logs missing transactions                       | Compare both transactions                        | Count Number of Missing Transactions   | Comparisons of all transactions: Logs missing transactions also to DataNotInsert sheet.                             |
| Finding Amount Difference       | Filter                       | Filters transactions with differing amounts   | Compare both transactions                        | Adding Amount difference transactions  | Comparisons of all transactions: Identifies amount differences for reconciliation.                                 |
| Adding Amount difference transactions | Google Sheets            | Logs amount difference transactions            | Finding Amount Difference                         | Count Number of Amount Difference Transactions | Comparisons of all transactions: Logs amount differences to AmountDifference sheet.                                |
| Count valid transactions        | Summarize                    | Counts valid transactions                       | Valid Transactions                               | Merge with all transactions count      | Comparisons of all transactions and calculate total count.                                                          |
| Count invalid transactions      | Summarize                    | Counts invalid transactions                     | Invalid transaction                              | Merge with all transactions count      | Comparisons of all transactions and calculate total count.                                                          |
| Count Number of Missing Transactions | Summarize               | Counts missing transactions                     | Add missing transaction to Sheet                 | Merge with all transactions count      | Comparisons of all transactions and calculate total count.                                                          |
| Count Number of Amount Difference Transactions. | Summarize         | Counts amount difference transactions          | Adding Amount difference transactions             | Merge with all transactions count      | Comparisons of all transactions and calculate total count.                                                          |
| Merge with all transactions count| Merge                       | Merges all counts into one dataset              | Count valid transactions, Count invalid transactions, Count Number of Amount Difference Transactions., Count Number of Missing Transactions | Code in JavaScript                     | Merge All Transaction Counts: Combines count results for summary.                                                   |
| Code in JavaScript              | Code (JavaScript)            | Aggregates counts into summary JSON object     | Merge with all transactions count                 | Adding the final count of transactions  | Merge All Transaction Counts: Creates final summary row with counts.                                                |
| Adding the final count of transactions | Google Sheets           | Writes final summary counts to summary sheet   | Code in JavaScript                               | Send report to reconciliation team     | Add final summary row and Send email report: Writes summary to Reconciliation Summary sheet.                        |
| Send report to reconciliation team | Email Send               | Sends email report with reconciliation summary | Adding the final count of transactions            | -                                     | Add final summary row and Send email report: Sends detailed report to finance team.                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a `Schedule Trigger` node to start the workflow on a desired interval or manual trigger.

2. **Add Google Sheets Node to Read Local Database Transactions:**  
   - Create a `Google Sheets` node named `Transactions from Local Database`.  
   - Set operation to "Read Rows" from your specific Google Sheet and tab containing local transactions.  
   - Authenticate via Google Service Account credentials.

3. **Add Google Sheets Node to Read Payment Gateway Transactions:**  
   - Create another `Google Sheets` node named `Transaction from Payment Gateway`.  
   - Read rows from the payment gateway Google Sheet and tab.  
   - Use the same or appropriate Google Service Account credentials.

4. **Add Compare Datasets Node:**  
   - Add a `Compare Datasets` node named `Compare both transactions`.  
   - Connect outputs of both Google Sheets nodes to this node (Local Database as Input A, Payment Gateway as Input B).  
   - Configure `mergeByFields` to compare on `PaymentRef`.  
   - This node splits outputs into 4 categories: matched, differing, only in A, only in B.

5. **Add Google Sheets Nodes for Logging:**  
   - Create four Google Sheets nodes for different result types:  
     - `Valid Transactions`: Append or update on a sheet named `RealData`. Map `PaymentRef` and `Amount`.  
     - `Invalid transaction`: Append or update on a sheet named `DataNotInsert`. Map `PaymentRef`.  
     - `Add missing transaction to Sheet`: Append or update on the same `DataNotInsert` sheet for missing transactions.  
     - `Adding Amount difference transactions`: Append or update on a sheet named `AmountDifference`. Map `PaymentRef`, `DBAmount` (Local DB amount), and `PortalAmount` (Payment Gateway amount).

6. **Add Filter Node for Amount Differences:**  
   - Insert a `Filter` node named `Finding Amount Difference` between the differing transactions output of the compare node and the `Adding Amount difference transactions` node.  
   - Configure filter to check if `Amount` from Local DB ≠ `Amount` from Payment Gateway.

7. **(Optional) Add Duplicate Detection:**  
   - Add a `Code` node named `Finding duplicate transactions` to scan Local Database transactions for duplicate `PaymentRef` values.  
   - Add a Google Sheets node `Adding duplicate transactions` to log duplicates.  
   - Connect Local Database transactions to the code node and output to the duplicates sheet.

8. **Add Summarize Nodes for Counting:**  
   - For each category (`Valid Transactions`, `Invalid transaction`, `Add missing transaction to Sheet`, `Adding Amount difference transactions`), add a `Summarize` node to count unique `PaymentRef` entries.  
   - Connect each logging node to its respective summarize node.

9. **Add Merge Node to Combine Counts:**  
   - Add a `Merge` node named `Merge with all transactions count` with 4 inputs.  
   - Connect all summarize nodes to this merge node.

10. **Add Code Node to Aggregate Counts:**  
    - Add a `Code` node named `Code in JavaScript`.  
    - Use JavaScript to construct a single JSON object consolidating counts of valid, invalid, missing, and amount difference transactions.

11. **Add Google Sheets Node for Final Summary:**  
    - Create a Google Sheets node named `Adding the final count of transactions`.  
    - Append or update the summary record in a sheet named `Reconciliation Summary`.  
    - Map JSON fields to sheet columns.

12. **Add Email Node to Send Report:**  
    - Create an `Email Send` node named `Send report to reconciliation team`.  
    - Configure SMTP credentials.  
    - Compose an email body referencing the summary JSON fields dynamically.  
    - Set recipients accordingly.

13. **Connect Nodes Appropriately:**  
    - Connect the `Schedule Trigger` to both Google Sheets data retrieval nodes.  
    - Connect these to the `Compare both transactions` node.  
    - Route outputs to logging and filtering nodes as described.  
    - Connect logging nodes to counting nodes.  
    - Merge counts, aggregate with code, write summary, and send email.

14. **Credential Setup:**  
    - Configure Google Service Account credentials with appropriate access to all Google Sheets used.  
    - Configure SMTP credentials for email node with valid sender and recipient emails.

15. **Final Checks:**  
    - Validate sheet names, IDs, and column mappings to your actual Google Sheets.  
    - Test the email node with a test recipient.  
    - Adjust the schedule trigger as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates reconciliation between Local Database and Payment Gateway transactions by comparing datasets, logging discrepancies, and sending summary reports. Setup requires Google Sheets service account and SMTP credentials. Replace Google Sheet IDs and email addresses before use. | Workflow Overview & Setup instructions contained in the main sticky note node titled "How it works".                                                                                 |
| The `Compare both transactions` node uses a powerful merge/compare feature to split datasets into matched, differing, and missing records automatically, simplifying reconciliation logic.                                                                                                   | Sticky Note near `Compare both transactions` node explains the merge/compare functionality in detail.                                                                                |
| Duplicate transaction detection is implemented but currently disabled, indicating optional use depending on business requirements.                                                                                                                                                            | Sticky Note near the duplicate detection nodes explains purpose and usage.                                                                                                           |
| The email report provides a clear summary of reconciliation results, facilitating quick review by finance teams and supports audit trails when combined with Google Sheets logging.                                                                                                         | Sticky Note near email and summary nodes highlights the reporting functionality.                                                                                                     |
| Ensure all Google Sheets tabs have correct headers and consistent data types (e.g., `PaymentRef` as string, `Amount` as numeric or string) to avoid comparison or mapping errors.                                                                                                             | General best practice when interacting with Google Sheets nodes.                                                                                                                    |
| For improved security, use environment variables or credential vaults for sensitive data like Google API and SMTP credentials.                                                                                                                                                                | Security best practices for workflow credential management.                                                                                                                         |
| Official n8n documentation on [Compare Datasets Node](https://docs.n8n.io/nodes/n8n-nodes-base.compareDatasets/) and [Google Sheets Node](https://docs.n8n.io/nodes/n8n-nodes-base.googleSheets/) can provide deeper insights on configuration options.                                      | n8n Docs                                                                                                                                                                            |

---

**Disclaimer:** The provided content is extracted solely from an n8n workflow JSON export, with all data and operations fully compliant with applicable content policies. No illegal or sensitive data is involved.