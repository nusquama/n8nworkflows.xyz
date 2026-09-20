Validate and reconcile CRM lead CSV imports with built-in tools

https://n8nworkflows.xyz/workflows/validate-and-reconcile-crm-lead-csv-imports-with-built-in-tools-17611


# Validate and reconcile CRM lead CSV imports with built-in tools

### 1. Workflow Overview

The **Validate and reconcile CRM lead CSV imports** workflow automates the ingestion, sanitization, validation, and reconciliation of lead data uploaded via a CSV file. Its primary purpose is to act as a quality gate before external lead data enters a production CRM, preventing duplicate records, formatting errors, and missing critical fields from corrupting the database.

The logical execution flows through the following functional blocks:
- **1.1 Input Reception & File Extraction:** Captures the source CSV file through an interactive n8n form and parses its raw binary content into structured JSON records.
- **1.2 Lead Preparation & Normalization:** Injects business quality rules, standardizes identity formats (names, emails, and phone numbers), and validates contactability against predefined criteria.
- **1.3 Reconciliation & Decision Routing:** Groups duplicate entries, merges complementary missing data while capturing conflicting field values, and routes the batch based on overall data health.
- **1.4 Output Packaging & Audit Delivery:** Formats the final outcome into an audit package ready for downstream consumption or manual review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Extraction
**Overview:** This block establishes the entry point of the workflow, providing a web form interface for user uploads and translating the uploaded binary CSV into actionable JSON objects.

- **Nodes Involved:**
  - `Upload source CSV`
  - `Extract CSV records`

- **Node Details:**
  - **Upload source CSV**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Form Trigger Node). Acts as the workflow webhook and interactive UI endpoint.
    - *Configuration Choices:* Configured with path `crm-lead-import-quality-gate` and a single mandatory file-type form field labeled "Lead CSV". Response mode set to `lastNode`.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Output connects to `Extract CSV records`.
    - *Version-Specific Requirements:* Version 2.6.
    - *Edge Cases/Failure Types:* File upload failures, incorrect file types (non-CSV), or payload size limits exceeded.
  - **Extract CSV records**
    - *Type and Technical Role:* `n8n-nodes-base.extractFromFile` (Extract from File Node). Parses binary file payloads into structured data rows.
    - *Configuration Choices:* Operation set to `csv` using the binary property name `data`.
    - *Key Expressions/Variables:* References binary property `data`.
    - *Input/Output Connections:* Input from `Upload source CSV`; output connects to `Configure lead quality rules`.
    - *Version-Specific Requirements:* Version 1.1.
    - *Edge Cases/Failure Types:* Malformed CSV syntax, missing headers, or unsupported character encodings.

---

#### 2.2 Lead Preparation & Normalization
**Overview:** This block injects governing business rules, trims and sanitizes text strings, standardizes telephone and email formats, and filters out uncontactable leads.

- **Nodes Involved:**
  - `Configure lead quality rules`
  - `Normalize lead identity fields`
  - `Validate lead contactability`

- **Node Details:**
  - **Configure lead quality rules**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields / Set Node). Defines global validation parameters and identity keys.
    - *Configuration Choices:* Mode set to `raw` with JSON output establishing `requiredFields` (`name`, `email`), `emailField`, `phoneField`, and `duplicateKeys` (`email`, `phone`). Includes other incoming fields.
    - *Key Expressions/Variables:* Explicit JSON configuration block.
    - *Input/Output Connections:* Input from `Extract CSV records`; output connects to `Normalize lead identity fields`.
    - *Version-Specific Requirements:* Version 3.4.
    - *Edge Cases/Failure Types:* JSON syntax errors in raw configuration payload.
  - **Normalize lead identity fields**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node - JavaScript). Cleans, trims, and formats raw strings across all incoming records.
    - *Configuration Choices:* Custom JavaScript processing array iteration to lowercase emails, trim whitespace from names and companies, and format telephone numbers to retain leading plus signs and digits only.
    - *Key Expressions/Variables:* Uses `$input.all()`.
    - *Input/Output Connections:* Input from `Configure lead quality rules`; output connects to `Validate lead contactability`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* Null or undefined record properties triggering type coercion errors.
  - **Validate lead contactability**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node - JavaScript). Evaluates mandatory fields and email syntax validity.
    - *Configuration Choices:* Custom JavaScript iterating over records to verify required fields and validate email structures via regular expression (`/^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$/`). Splits data into `valid` and `rejected` arrays.
    - *Key Expressions/Variables:* Evaluates item properties and `requiredFields` configuration.
    - *Input/Output Connections:* Input from `Normalize lead identity fields`; output connects to `Reconcile duplicate leads`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* Empty input arrays or non-standard email structures.

---

#### 2.3 Reconciliation & Decision Routing
**Overview:** This block analyzes valid records for duplicate identities, merges complementary missing data, flags conflicting attribute values, and routes the batch based on data hygiene status.

- **Nodes Involved:**
  - `Reconcile duplicate leads`
  - `Check result status`

- **Node Details:**
  - **Reconcile duplicate leads**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node - JavaScript). Groups duplicate records by email/phone keys, fills in missing attributes, logs data conflicts, and compiles summary metrics.
    - *Configuration Choices:* Custom JavaScript mapping groups and determining overall batch status (`ready` or `review_required`).
    - *Key Expressions/Variables:* Evaluates item records, accepted sets, and conflict reasons.
    - *Input/Output Connections:* Input from `Validate lead contactability`; output connects to `Check result status`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* Memory overhead with excessively large CSV files.
  - **Check result status**
    - *Type and Technical Role:* `n8n-nodes-base.if` (If Node). Branching gateway that evaluates the workflow's compliance status.
    - *Configuration Choices:* Strict type validation checking if `{{ $json.status === 'ready' }}` is true.
    - *Key Expressions/Variables:* `={{ $json.status === 'ready' }}`
    - *Input/Output Connections:* Input from `Reconcile duplicate leads`; outputs connect to `Package import-ready lead batch` (true branch) and `Package lead review report` (false branch).
    - *Version-Specific Requirements:* Version 2.3.
    - *Edge Cases/Failure Types:* Unexpected status string values causing conditional fallback.

---

#### 2.4 Output Packaging & Audit Delivery
**Overview:** This block applies outcome markers to the processed data packages, merges the conditional branches back into a single pipeline, and outputs the final structured audit report.

- **Nodes Involved:**
  - `Package import-ready lead batch`
  - `Package lead review report`
  - `Merge result route`
  - `Return CRM lead audit`

- **Node Details:**
  - **Package import-ready lead batch**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node - JavaScript). Appends an explicit readiness outcome tag.
    - *Configuration Choices:* Maps input items adding `outcome: 'ready'`.
    - *Key Expressions/Variables:* `$input.all()`
    - *Input/Output Connections:* Input from `Check result status` (true branch); output connects to `Merge result route`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* None specific.
  - **Package lead review report**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node - JavaScript). Appends a review-required outcome tag.
    - *Configuration Choices:* Maps input items adding `outcome: 'review_required'`.
    - *Key Expressions/Variables:* `$input.all()`
    - *Input/Output Connections:* Input from `Check result status` (false branch); output connects to `Merge result route`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* None specific.
  - **Merge result route**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Merge Node). Combines parallel execution paths back into a single flow.
    - *Configuration Choices:* Mode set to `append` with 2 input streams.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Inputs from both packaging nodes; output connects to `Return CRM lead audit`.
    - *Version-Specific Requirements:* Version 3.2.
    - *Edge Cases/Failure Types:* Branch synchronization mismatches.
  - **Return CRM lead audit**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node - JavaScript). Finalizes and passes through the complete audit payload.
    - *Configuration Choices:* Returns `$input.all()`.
    - *Key Expressions/Variables:* `$input.all()`
    - *Input/Output Connections:* Input from `Merge result route`; output terminates workflow execution.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* Large payload size returning to webhook response.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Upload source CSV | n8n-nodes-base.formTrigger | Accepts lead CSV uploads via web form | None | Extract CSV records | Use this workflow when event, partner, or legacy-system lead exports must be checked before anyone imports them into a CRM. It turns a raw CSV into one audit package containing accepted rows, rejected rows, duplicate counts, and source-attributed field conflicts.<br><br>### How it works<br><br>The form accepts a CSV, the visible configuration declares required and identity fields, and three processing stages normalize contact data, validate contactability, and reconcile duplicates. Complementary empty fields are filled, but competing non-empty values are never silently overwritten.<br><br>### Setup<br><br>Open Configure lead quality rules and match the field names to your export. Upload a representative file, review the rejected reason codes and conflicts, then use only the accepted collection for a downstream CRM import.<br><br>The workflow uses built-in credential-free nodes. Email checks validate format only, and phone cleanup does not verify carrier reachability. |
| Extract CSV records | n8n-nodes-base.extractFromFile | Parses CSV binary data into JSON records | Upload source CSV | Configure lead quality rules | Prepare and validate leads<br><br>Upload the CSV, configure field policy, normalize identity fields, and reject rows that cannot be contacted. |
| Configure lead quality rules | n8n-nodes-base.set | Defines required fields and duplicate keys | Extract CSV records | Normalize lead identity fields | Prepare and validate leads<br><br>Upload the CSV, configure field policy, normalize identity fields, and reject rows that cannot be contacted. |
| Normalize lead identity fields | n8n-nodes-base.code | Trims and normalizes name, email, and phone formats | Configure lead quality rules | Validate lead contactability | Prepare and validate leads<br><br>Upload the CSV, configure field policy, normalize identity fields, and reject rows that cannot be contacted. |
| Validate lead contactability | n8n-nodes-base.code | Validates required fields and checks email syntax | Normalize lead identity fields | Reconcile duplicate leads | Prepare and validate leads<br><br>Upload the CSV, configure field policy, normalize identity fields, and reject rows that cannot be contacted. |
| Reconcile duplicate leads | n8n-nodes-base.code | Merges duplicate profiles and identifies conflicts | Validate lead contactability | Check result status | Reconcile and review<br><br>Merge complementary duplicates, preserve disagreements as conflicts, and return one explicit import decision. |
| Check result status | n8n-nodes-base.if | Routes batch based on presence of rejects or conflicts | Reconcile duplicate leads | Package import-ready lead batch, Package lead review report | Reconcile and review<br><br>Merge complementary duplicates, preserve disagreements as conflicts, and return one explicit import decision. |
| Package import-ready lead batch | n8n-nodes-base.code | Tags batch outcome as ready | Check result status | Merge result route | Reconcile and review<br><br>Merge complementary duplicates, preserve disagreements as conflicts, and return one explicit import decision. |
| Package lead review report | n8n-nodes-base.code | Tags batch outcome as review required | Check result status | Merge result route | Reconcile and review<br><br>Merge complementary duplicates, preserve disagreements as conflicts, and return one explicit import decision. |
| Merge result route | n8n-nodes-base.merge | Combines conditional execution paths | Package import-ready lead batch, Package lead review report | Return CRM lead audit | Reconcile and review<br><br>Merge complementary duplicates, preserve disagreements as conflicts, and return one explicit import decision. |
| Return CRM lead audit | n8n-nodes-base.code | Outputs final audit package | Merge result route | None | Reconcile and review<br><br>Merge complementary duplicates, preserve disagreements as conflicts, and return one explicit import decision. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create Node 1: Upload source CSV**
   - **Type:** `n8n-nodes-base.formTrigger`
   - **Parameters:** Set path to `crm-lead-import-quality-gate`. Add a form field with field type `file`, label `Lead CSV`, and mark it as required. Set response mode to `lastNode`.
   - **Credentials:** None required.

2. **Create Node 2: Extract CSV records**
   - **Type:** `n8n-nodes-base.extractFromFile`
   - **Parameters:** Set operation to `csv` and binary property name to `data`.
   - **Connection:** Connect output of `Upload source CSV` to this node.

3. **Create Node 3: Configure lead quality rules**
   - **Type:** `n8n-nodes-base.set`
   - **Parameters:** Set mode to `raw`. Provide JSON output configuring `requiredFields` (`name`, `email`), `emailField`, `phoneField`, and `duplicateKeys` (`email`, `phone`). Ensure "Include Other Fields" is enabled.
   - **Connection:** Connect output of `Extract CSV records` to this node.

4. **Create Node 4: Normalize lead identity fields**
   - **Type:** `n8n-nodes-base.code`
   - **Parameters:** Insert JavaScript to map and sanitize `name`, lowercase `email`, format `phone` digits, and trim `company` fields across records.
   - **Connection:** Connect output of `Configure lead quality rules` to this node.

5. **Create Node 5: Validate lead contactability**
   - **Type:** `n8n-nodes-base.code`
   - **Parameters:** Insert JavaScript evaluating required fields and validating email patterns against regex, sorting records into `valid` and `rejected` arrays with reason codes.
   - **Connection:** Connect output of `Normalize lead identity fields` to this node.

6. **Create Node 6: Reconcile duplicate leads**
   - **Type:** `n8n-nodes-base.code`
   - **Parameters:** Insert JavaScript grouping items by duplicate keys, merging empty complementary fields, registering field disagreements as conflicts, and determining overall batch status (`ready` or `review_required`).
   - **Connection:** Connect output of `Validate lead contactability` to this node.

7. **Create Node 7: Check result status**
   - **Type:** `n8n-nodes-base.if`
   - **Parameters:** Set condition type validation to strict, checking expression `={{ $json.status === 'ready' }}` for boolean true.
   - **Connection:** Connect output of `Reconcile duplicate leads` to this node.

8. **Create Node 8: Package import-ready lead batch**
   - **Type:** `n8n-nodes-base.code`
   - **Parameters:** Insert JavaScript returning items with `outcome: 'ready'`.
   - **Connection:** Connect true output of `Check result status` to this node.

9. **Create Node 9: Package lead review report**
   - **Type:** `n8n-nodes-base.code`
   - **Parameters:** Insert JavaScript returning items with `outcome: 'review_required'`.
   - **Connection:** Connect false output of `Check result status` to this node.

10. **Create Node 10: Merge result route**
    - **Type:** `n8n-nodes-base.merge`
    - **Parameters:** Set mode to `append` with 2 input streams.
    - **Connection:** Connect outputs of both `Package import-ready lead batch` and `Package lead review report` to inputs 1 and 2 of this node.

11. **Create Node 11: Return CRM lead audit**
    - **Type:** `n8n-nodes-base.code`
    - **Parameters:** Insert JavaScript returning `$input.all()`.
    - **Connection:** Connect output of `Merge result route` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built-in credential-free architecture utilizing standard n8n expression and code execution nodes. | Workflow design pattern |
| Email validation enforces structural syntax formatting only; phone normalization strips non-digit characters without validating carrier reachability. | Data validation constraints |