Validate DigiParser purchase orders and export staging CSV files

https://n8nworkflows.xyz/workflows/validate-digiparser-purchase-orders-and-export-staging-csv-files-20435


# Validate DigiParser purchase orders and export staging CSV files

### 1. Workflow Overview

This workflow validates DigiParser purchase order extraction events, separates valid orders from problematic ones based on rigorous quality and arithmetic rules, and exports two distinct CSV files: one containing staging-ready line items for downstream ERP import, and another containing flagged orders requiring manual review.

The logical execution is organized into the following blocks:
- **1.1 Input Reception & Simulation:** Triggers the workflow execution and loads synthetic purchase order extraction events to simulate an incoming data stream.
- **1.2 Data Validation & Integrity Checking:** Evaluates the structure, required fields, formats, financial calculations, and intra-batch uniqueness of the purchase order payloads.
- **1.3 Conditional Routing & Data Transformation:** Branches the execution path depending on validation outcomes and flattens or formats the data structures into tabular schemas.
- **1.4 File Generation & Export:** Converts the processed datasets into downloadable CSV files for operational staging and review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Simulation
- **Overview:** Initializes the workflow execution manually and provides a controlled set of mock purchase order payloads representing various validation scenarios (valid, total mismatch, and duplicate).
- **Nodes Involved:** `Run demo`, `Load synthetic DigiParser events`.
- **Node Details:**
  - **Run demo**
    - Type: `n8n-nodes-base.manualTrigger` (Version 1)
    - Technical Role: Acts as the entry point for manual workflow execution.
    - Configuration Choices: No parameters required.
    - Input Connections: None (Trigger node).
    - Output Connections: Connects to `Load synthetic DigiParser events`.
    - Edge Cases: Designed solely for testing; in production, this should be replaced with a webhook or API trigger.
  - **Load synthetic DigiParser events**
    - Type: `n8n-nodes-base.code` (Version 2)
    - Technical Role: Generates an array of three static JSON objects emulating DigiParser `document.exported` webhook events.
    - Configuration Choices: Uses custom JavaScript to return an array of JSON payloads containing header details and line items.
    - Input Connections: Receives execution signal from `Run demo`.
    - Output Connections: Connects to `Validate customer orders`.
    - Edge Cases: Synthetic data is limited to one batch context; real-world integrations require proper payload mapping.

#### 2.2 Data Validation & Integrity Checking
- **Overview:** Inspects every incoming purchase order for required headers, correct currency formatting, valid calendar dates, positive line quantities, internal arithmetic precision, order total reconciliation, and intra-batch duplicate detection.
- **Nodes Involved:** `Validate customer orders`.
- **Node Details:**
  - **Validate customer orders**
    - Type: `n8n-nodes-base.code` (Version 2)
    - Technical Role: Executes comprehensive data quality and business logic checks on all incoming items.
    - Configuration Choices: Iterates over all items using a custom script (`$input.all()`), applies monetary and string sanitization helpers, tracks seen keys for duplicate detection, and populates a `review_reasons` array if errors are detected.
    - Key Expressions / Variables: `{{ $json.body ?? $json }}`, `validation_status`, `valid`, `review_reasons`.
    - Input Connections: Receives data from `Load synthetic DigiParser events`.
    - Output Connections: Connects to `Order passes checks?`.
    - Edge Cases: Fails if unexpected data structures or null values bypass structural checks; duplicate detection is strictly scoped to the active execution batch.

#### 2.3 Conditional Routing & Data Transformation
- **Overview:** Splits the validated dataset based on the `valid` boolean flag and reshapes the record structures into flattened tabular formats suited for CSV export.
- **Nodes Involved:** `Order passes checks?`, `Prepare ready order rows`, `Prepare review rows`.
- **Node Details:**
  - **Order passes checks?**
    - Type: `n8n-nodes-base.if` (Version 2.2)
    - Technical Role: Evaluates whether an order passed all validation rules.
    - Configuration Choices: Checks if `{{ $json.valid }}` evaluates to `true`.
    - Input Connections: Receives items from `Validate customer orders`.
    - Output Connections: 
        - True branch connects to `Prepare ready order rows`.
        - False branch connects to `Prepare review rows`.
    - Edge Cases: Malformed boolean values or missing parameters will route items to the review branch.
  - **Prepare ready order rows**
    - Type: `n8n-nodes-base.code` (Version 2)
    - Technical Role: Flattens valid purchase orders from a header-level structure with nested line items into individual line-level rows.
    - Configuration Choices: Iterates through each line item of valid orders, appending header metadata to each line-item object.
    - Input Connections: Receives true-branch items from `Order passes checks?`.
    - Output Connections: Connects to `Download staging CSV`.
    - Edge Cases: Orders containing empty line-item arrays will yield no output rows.
  - **Prepare review rows**
    - Type: `n8n-nodes-base.code` (Version 2)
    - Technical Role: Formats invalid orders and their corresponding error messages for operational inspection.
    - Configuration Choices: Maps error reasons from an array into a single semicolon-separated string (`review_reasons.join('; ')`).
    - Input Connections: Receives false-branch items from `Order passes checks?`.
    - Output Connections: Connects to `Download review CSV`.
    - Edge Cases: Unhandled exception objects might result in empty error strings if not properly converted.

#### 2.4 File Generation & Export
- **Overview:** Converts the structured JavaScript objects into downloadable CSV binary files containing ready-to-stage orders or orders needing review.
- **Nodes Involved:** `Download staging CSV`, `Download review CSV`.
- **Node Details:**
  - **Download staging CSV**
    - Type: `n8n-nodes-base.convertToFile` (Version 1.1)
    - Technical Role: Converts tabular line-item objects into a CSV file format.
    - Configuration Choices: 
        - Operation: `csv`
        - File Name: `ready-orders.csv`
        - Header Row: Enabled (`true`)
    - Input Connections: Receives transformed rows from `Prepare ready order rows`.
    - Output Connections: None (Terminal node).
    - Edge Cases: Large datasets may consume significant memory during conversion.
  - **Download review CSV**
    - Type: `n8n-nodes-base.convertToFile` (Version 1.1)
    - Technical Role: Converts invalid order exception records into a CSV file format.
    - Configuration Choices: 
        - Operation: `csv`
        - File Name: `review-orders.csv`
        - Header Row: Enabled (`true`)
    - Input Connections: Receives transformed rows from `Prepare review rows`.
    - Output Connections: None (Terminal node).
    - Edge Cases: Generates an empty file structure if no orders fail validation.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run demo | n8n-nodes-base.manualTrigger | Trigger manual workflow run | None | Load synthetic DigiParser events | Validate customer purchase orders and export staging CSV files<br><br>Who is this for?<br>Seller-side order-entry teams processing customer purchase orders with DigiParser. Founder affiliation: contributed by DigiParser.<br><br>Run the demo<br>Import and run Run demo. Three synthetic document.exported events demonstrate a correct order, a total mismatch and a duplicate. The correct order produces two line-item CSV rows; the review file contains two cases. No account connection is needed for this demo.<br><br>Use your extracted data<br>Replace Load synthetic DigiParser events with a trusted data source. Each input item must contain event: document.exported and data.id plus data.extracted_data. For REST API output, map documentId to data.id, combine fields with tables into data.extracted_data, and preserve the line-item field names shown in the demo. Use an authenticated API request or a signature-verified webhook; never expose an unsigned public order-entry endpoint. Configure your credentials in n8n, not in this JSON.<br><br>What it does<br>Checks required fields, dates, positive quantities, decimal money, line arithmetic and order total. Detects repeated customer+PO pairs within one batch. Routes exceptions to review and writes CSV files for inspection. Does not write an ERP order.<br><br>Assumptions<br>Two-decimal currency; total equals line totals without tax, freight or discounts. Add those components for other order formats. Duplicate checks do not cover retries across runs.<br><br>Setup context: https://www.digiparser.com/solutions/purchase-order-parser<br>API and webhook docs: https://www.digiparser.com/docs/api/webhooks |
| Load synthetic DigiParser events | n8n-nodes-base.code | Load mock payload data | Run demo | Validate customer orders | Validate customer purchase orders and export staging CSV files<br><br>Who is this for?<br>Seller-side order-entry teams processing customer purchase orders with DigiParser. Founder affiliation: contributed by DigiParser.<br><br>Run the demo<br>Import and run Run demo. Three synthetic document.exported events demonstrate a correct order, a total mismatch and a duplicate. The correct order produces two line-item CSV rows; the review file contains two cases. No account connection is needed for this demo.<br><br>Use your extracted data<br>Replace Load synthetic DigiParser events with a trusted data source. Each input item must contain event: document.exported and data.id plus data.extracted_data. For REST API output, map documentId to data.id, combine fields with tables into data.extracted_data, and preserve the line-item field names shown in the demo. Use an authenticated API request or a signature-verified webhook; never expose an unsigned public order-entry endpoint. Configure your credentials in n8n, not in this JSON.<br><br>What it does<br>Checks required fields, dates, positive quantities, decimal money, line arithmetic and order total. Detects repeated customer+PO pairs within one batch. Routes exceptions to review and writes CSV files for inspection. Does not write an ERP order.<br><br>Assumptions<br>Two-decimal currency; total equals line totals without tax, freight or discounts. Add those components for other order formats. Duplicate checks do not cover retries across runs.<br><br>Setup context: https://www.digiparser.com/solutions/purchase-order-parser<br>API and webhook docs: https://www.digiparser.com/docs/api/webhooks<br><br>Demo source<br>Three synthetic export events: a valid order, a total mismatch and a same-batch duplicate. Replace this Code node with a trusted, mapped source for real use. |
| Validate customer orders | n8n-nodes-base.code | Validate order fields and rules | Load synthetic DigiParser events | Order passes checks? | Validate customer purchase orders and export staging CSV files<br><br>Who is this for?<br>Seller-side order-entry teams processing customer purchase orders with DigiParser. Founder affiliation: contributed by DigiParser.<br><br>Run the demo<br>Import and run Run demo. Three synthetic document.exported events demonstrate a correct order, a total mismatch and a duplicate. The correct order produces two line-item CSV rows; the review file contains two cases. No account connection is needed for this demo.<br><br>Use your extracted data<br>Replace Load synthetic DigiParser events with a trusted data source. Each input item must contain event: document.exported and data.id plus data.extracted_data. For REST API output, map documentId to data.id, combine fields with tables into data.extracted_data, and preserve the line-item field names shown in the demo. Use an authenticated API request or a signature-verified webhook; never expose an unsigned public order-entry endpoint. Configure your credentials in n8n, not in this JSON.<br><br>What it does<br>Checks required fields, dates, positive quantities, decimal money, line arithmetic and order total. Detects repeated customer+PO pairs within one batch. Routes exceptions to review and writes CSV files for inspection. Does not write an ERP order.<br><br>Assumptions<br>Two-decimal currency; total equals line totals without tax, freight or discounts. Add those components for other order formats. Duplicate checks do not cover retries across runs.<br><br>Setup context: https://www.digiparser.com/solutions/purchase-order-parser<br>API and webhook docs: https://www.digiparser.com/docs/api/webhooks<br><br>Check order data<br>Required fields, calendar date, two-decimal money, positive quantities, line arithmetic and total reconciliation. Duplicate detection is limited to this batch. |
| Order passes checks? | n8n-nodes-base.if | Route orders based on validity | Validate customer orders | Prepare ready order rows, Prepare review rows | |
| Prepare ready order rows | n8n-nodes-base.code | Flatten valid orders into line items | Order passes checks? | Download staging CSV | |
| Prepare review rows | n8n-nodes-base.code | Format invalid orders for review | Order passes checks? | Download review CSV | |
| Demo setup and scope | n8n-nodes-base.stickyNote | Workflow overview and setup instructions | None | None | Validate customer purchase orders and export staging CSV files<br><br>Who is this for?<br>Seller-side order-entry teams processing customer purchase orders with DigiParser. Founder affiliation: contributed by DigiParser.<br><br>Run the demo<br>Import and run Run demo. Three synthetic document.exported events demonstrate a correct order, a total mismatch and a duplicate. The correct order produces two line-item CSV rows; the review file contains two cases. No account connection is needed for this demo.<br><br>Use your extracted data<br>Replace Load synthetic DigiParser events with a trusted data source. Each input item must contain event: document.exported and data.id plus data.extracted_data. For REST API output, map documentId to data.id, combine fields with tables into data.extracted_data, and preserve the line-item field names shown in the demo. Use an authenticated API request or a signature-verified webhook; never expose an unsigned public order-entry endpoint. Configure your credentials in n8n, not in this JSON.<br><br>What it does<br>Checks required fields, dates, positive quantities, decimal money, line arithmetic and order total. Detects repeated customer+PO pairs within one batch. Routes exceptions to review and writes CSV files for inspection. Does not write an ERP order.<br><br>Assumptions<br>Two-decimal currency; total equals line totals without tax, freight or discounts. Add those components for other order formats. Duplicate checks do not cover retries across runs.<br><br>Setup context: https://www.digiparser.com/solutions/purchase-order-parser<br>API and webhook docs: https://www.digiparser.com/docs/api/webhooks |
| Download staging CSV | n8n-nodes-base.convertToFile | Export valid lines to CSV | Prepare ready order rows | None | Inspect both files<br>Valid orders become line-level staging rows. Exceptions include reasons in the review file. Download the binary output from each final node. No ERP writes occur. |
| Download review CSV | n8n-nodes-base.convertToFile | Export review items to CSV | Prepare review rows | None | Inspect both files<br>Valid orders become line-level staging rows. Exceptions include reasons in the review file. Download the binary output from each final node. No ERP writes occur. |
| Demo data source | n8n-nodes-base.stickyNote | Notes for the synthetic data node | None | None | Demo source<br>Three synthetic export events: a valid order, a total mismatch and a same-batch duplicate. Replace this Code node with a trusted, mapped source for real use. |
| Validation checks | n8n-nodes-base.stickyNote | Notes for validation rules | None | None | Check order data<br>Required fields, calendar date, two-decimal money, positive quantities, line arithmetic and total reconciliation. Duplicate detection is limited to this batch. |
| output-note | n8n-nodes-base.stickyNote | Notes for CSV outputs | None | None | Inspect both files<br>Valid orders become line-level staging rows. Exceptions include reasons in the review file. Download the binary output from each final node. No ERP writes occur. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run demo`. Leave parameters empty.
2. **Add the Synthetic Data Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Load synthetic DigiParser events` (Version 2).
   - Insert the JavaScript code returning the three synthetic JSON payloads (`demo_doc_001`, `demo_doc_002`, and a duplicate `demo_doc_001`).
   - Connect `Run demo`'s output to `Load synthetic DigiParser events`.
3. **Add the Validation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate customer orders` (Version 2).
   - Insert the validation logic script that verifies required properties, currencies, dates, money formats, line arithmetic, and duplicate tracking.
   - Connect `Load synthetic DigiParser events`'s output to `Validate customer orders`.
4. **Add the Conditional Router Node:**
   - Add an **If** node (`n8n-nodes-base.if`, Version 2.2). Name it `Order passes checks?`.
   - Configure a condition where `{{ $json.valid }}` equals `true` (boolean operation).
   - Connect `Validate customer orders`'s output to `Order passes checks?`.
5. **Add the Staging Transformation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Prepare ready order rows` (Version 2).
   - Insert the flattening script that extracts line items from valid orders into individual tabular rows.
   - Connect the `true` output of `Order passes checks?` to `Prepare ready order rows`.
6. **Add the Review Transformation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Prepare review rows` (Version 2).
   - Insert the script mapping error reasons into a semicolon-separated string for invalid orders.
   - Connect the `false` output of `Order passes checks?` to `Prepare review rows`.
7. **Add the Staging CSV Export Node:**
   - Add a **Convert to File** node (`n8n-nodes-base.convertToFile`, Version 1.1). Name it `Download staging CSV`.
   - Set Operation to `csv`, File Name to `ready-orders.csv`, and Header Row to `true`.
   - Connect `Prepare ready order rows`'s output to `Download staging CSV`.
8. **Add the Review CSV Export Node:**
   - Add a **Convert to File** node (`n8n-nodes-base.convertToFile`, Version 1.1). Name it `Download review CSV`.
   - Set Operation to `csv`, File Name to `review-orders.csv`, and Header Row to `true`.
   - Connect `Prepare review rows`'s output to `Download review CSV`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| DigiParser Purchase Order Solution | [DigiParser Solutions Context](https://www.digiparser.com/solutions/purchase-order-parser) |
| API and Webhook Documentation | [DigiParser Webhook Docs](https://www.digiparser.com/docs/api/webhooks) |