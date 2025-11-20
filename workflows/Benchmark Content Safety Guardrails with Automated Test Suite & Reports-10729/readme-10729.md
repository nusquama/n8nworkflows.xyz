Benchmark Content Safety Guardrails with Automated Test Suite & Reports

https://n8nworkflows.xyz/workflows/benchmark-content-safety-guardrails-with-automated-test-suite---reports-10729


# Benchmark Content Safety Guardrails with Automated Test Suite & Reports

### 1. Workflow Overview

This workflow benchmarks and evaluates the effectiveness of the n8n **Guardrails node** in detecting unsafe or non-compliant content using a curated automated test suite. It processes 36 predefined test cases representing a variety of safety-sensitive categories such as Jailbreak, NSFW, Personally Identifiable Information (PII), Keywords, URLs, and Secret Keys. The workflow runs these cases through Guardrails, classifies the results into PASS or VIOLATION, calculates detailed accuracy metrics (confusion matrix, precision, recall, F1 score), generates a comprehensive Markdown/HTML report, and emails the report to a specified inbox.

The workflow is logically grouped into the following blocks:

- 1.1 Input Reception and Test Data Initialization  
- 1.2 Iterative Processing Loop for Guardrails Evaluation  
- 1.3 Preservation and Guardrails Analysis  
- 1.4 Result Normalization and Classification  
- 1.5 Result Aggregation and Metrics Calculation  
- 1.6 Report Generation and Delivery  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Test Data Initialization

- **Overview:** Accepts manual trigger input and initializes the 36 predefined test cases for evaluation.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Set Test Data (code)

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for workflow execution; user-initiated start.  
    - Configuration: No parameters; simply initiates workflow on user command.  
    - Inputs: None  
    - Outputs: Connects to "Set Test Data (code)"  
    - Edge Cases: No input, so minimal failure risk; ensure user triggers workflow.  

  - **Set Test Data (code)**  
    - Type: Code (JavaScript)  
    - Role: Generates an array of 36 structured test cases with ID, category, text, expected outcome, and description.  
    - Configuration: Hardcoded test cases covering multiple content safety categories and edge cases.  
    - Expressions: Returns array mapped to individual JSON items (one per test case).  
    - Inputs: Manual Trigger node output  
    - Outputs: Feeds into Loop Over Items node  
    - Edge Cases: Care needed if test data is modified; must maintain expected structure for downstream matching.

---

#### 1.2 Iterative Processing Loop for Guardrails Evaluation

- **Overview:** Iterates sequentially over each test case, to process one item at a time for isolated evaluation.
- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)

- **Node Details:**  
  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes each test case individually to prevent cross-contamination of results and facilitate matching of outputs to inputs.  
    - Configuration: Default batch size (1 item per batch).  
    - Inputs: Test cases from "Set Test Data (code)"  
    - Outputs: Two outputs: main for accumulating processed data, and secondary for preserving original data.  
    - Edge Cases: Large batch size may cause data mismatch; default 1 ensures stable processing.

---

#### 1.3 Preservation and Guardrails Analysis

- **Overview:** Preserves original metadata from each test case to prevent data loss during Guardrails processing, then applies Guardrails checks to evaluate content safety.
- **Nodes Involved:**  
  - Preserve Original Data (Code)  
  - Check Guardrails (Guardrails Node)

- **Node Details:**  
  - **Preserve Original Data**  
    - Type: Code (JavaScript)  
    - Role: Extracts and flattens key test metadata (_id, _category, _expected, _description) alongside the text, so the Guardrails node does not overwrite these fields.  
    - Configuration: Returns JSON with top-level fields, including required "text" for Guardrails.  
    - Inputs: Loop Over Items output (individual test case)  
    - Outputs: Feeds into "Check Guardrails"  
    - Edge Cases: If Guardrails input format changes, preservation may fail or metadata can be lost.  

  - **Check Guardrails**  
    - Type: Guardrails Node (n8n-nodes-langchain.guardrails)  
    - Role: Core evaluation engine applying multiple safety filters including PII, NSFW (threshold 0.7), URLs, keywords ("illegal, drugs, hack, exploit, weapon"), jailbreak (threshold 0.7), and secret keys (balanced permissiveness).  
    - Configuration: Static guardrail settings as above; input text from preserved data.  
    - Inputs: Preserved Original Data output  
    - Outputs: Two outputs:  
      - Pass output for non-violations  
      - Fail output for violations  
    - Credentials: OpenAI API (for underlying LLM processing)  
    - Edge Cases: Possible failures include API auth errors, network timeouts, or unexpected Guardrails output structure.

---

#### 1.4 Result Normalization and Classification

- **Overview:** Converts Guardrails output (pass/fail) into a unified data structure with detailed classification, including expected vs. actual results, correctness, confidence score, violation types, and timestamps.
- **Nodes Involved:**  
  - Format Pass Result (Code)  
  - Format Fail Result (Code)

- **Node Details:**  
  - **Format Pass Result**  
    - Type: Code (JavaScript)  
    - Role: Formats a passing check as a standardized JSON item with status "PASS", matching original metadata by text content, and sets score to 0.  
    - Configuration: Attempts to match the current item to preserved metadata by text; falls back to last preserved item or current item fields.  
    - Inputs: Guardrails Pass output  
    - Outputs: Connects to "Combine Result" main input (index 0)  
    - Edge Cases: Failure to match preserved data can cause "unknown" fields; unexpected input structure may break matching logic.  

  - **Format Fail Result**  
    - Type: Code (JavaScript)  
    - Role: Formats a failed check as a standardized JSON item with status "VIOLATION", extracts triggered violation types and max confidence score, matches to preserved metadata.  
    - Configuration: Similar matching logic to pass formatter; extracts violation details from Guardrails "checks" array.  
    - Inputs: Guardrails Fail output  
    - Outputs: Connects to "Combine Result" secondary input (index 1)  
    - Edge Cases: If Guardrails "checks" missing or malformed, violation type or score may be inaccurate.

---

#### 1.5 Result Aggregation and Metrics Calculation

- **Overview:** Merges passing and violation results maintaining order, then calculates comprehensive accuracy metrics including confusion matrix, precision, recall, F1 score, per-category accuracy, and consolidated results.
- **Nodes Involved:**  
  - Combine Result (Merge)  
  - Calculate Metrics (Code)

- **Node Details:**  
  - **Combine Result**  
    - Type: Merge  
    - Role: Combines the two parallel streams of pass and fail formatted results into a single unified stream, preserving the original sequence.  
    - Configuration: Default merge mode (likely "Wait" or "Append") to unify inputs.  
    - Inputs: Outputs from both Format Pass Result and Format Fail Result nodes  
    - Outputs: Connects to "Loop Over Items" done output  
    - Edge Cases: Out-of-order merges can corrupt metrics; ensure ordered merging.  

  - **Calculate Metrics**  
    - Type: Code (JavaScript)  
    - Role: Processes the merged results to compute:  
      - Confusion matrix (true positives, true negatives, false positives, false negatives)  
      - Accuracy, precision, recall, F1 score  
      - Category-level accuracy and counts  
      - Prepares summary object and detailed results array for reporting  
    - Configuration: Iterates over all results, compares expected vs actual, aggregates statistics, calculates percentages, formats output JSON.  
    - Inputs: Merged results from "Loop Over Items" done output  
    - Outputs: Connects to "Format Report"  
    - Edge Cases: Empty input arrays yield zeroed metrics; data inconsistencies may lead to inaccurate stats.

---

#### 1.6 Report Generation and Delivery

- **Overview:** Generates a detailed Markdown report from metrics and results, converts it to HTML for presentation, then sends it by email.
- **Nodes Involved:**  
  - Format Report (Code)  
  - Markdown (Markdown)  
  - Send a message (Gmail)

- **Node Details:**  
  - **Format Report**  
    - Type: Code (JavaScript)  
    - Role: Builds a comprehensive Markdown report including:  
      - Overall metrics summary  
      - Advanced metrics (precision, recall, F1)  
      - Confusion matrix details  
      - Category performance breakdown  
      - Detailed table of test results with ID, category, expected vs actual, correctness, score, violation types  
      - Recommendations based on false positives/negatives and accuracy  
    - Inputs: Metrics summary and detailed results from "Calculate Metrics"  
    - Outputs: Connects to "Markdown" for Markdown-to-HTML conversion  
    - Edge Cases: Large result sets may produce long reports; malformed data may cause formatting issues.  

  - **Markdown**  
    - Type: Markdown  
    - Role: Converts Markdown-formatted report into HTML with emoji and table support for rich email presentation.  
    - Configuration: Markdown to HTML mode enabled with options for emoji and tables.  
    - Inputs: Markdown report from "Format Report"  
    - Outputs: Connects to "Send a message"  
    - Edge Cases: Complex Markdown may not render perfectly.  

  - **Send a message**  
    - Type: Gmail (OAuth2)  
    - Role: Sends the final HTML report as an email to the specified recipient.  
    - Configuration:  
      - Recipient email must be replaced manually at parameter `sendTo` (currently "YOUR_MAIL_HERE")  
      - Subject: "Guardrails Evaluation Report"  
      - Message body: HTML report content from Markdown node  
    - Inputs: HTML content from "Markdown"  
    - Credentials: Gmail OAuth2 credentials required  
    - Edge Cases: Email sending may fail due to auth errors, quota limits, or invalid recipient address.

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                                   | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                                  |
|----------------------------|-------------------------------|--------------------------------------------------|----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Entry point; starts workflow                      | None                             | Set Test Data (code)              |                                                                                                              |
| Set Test Data (code)        | Code                          | Generates 36 predefined test cases                | When clicking ‘Execute workflow’ | Loop Over Items                  | ## 📝 Test Case Generator  Creates 36 pre-defined prompts covering all safety categories.                    |
| Loop Over Items             | SplitInBatches                | Iterates over each test case sequentially         | Set Test Data (code)             | Calculate Metrics, Preserve Original Data | ## 🔁 Loop Through Each Test Case  Iterates 36 prompts; ensures clean per-item processing.                   |
| Preserve Original Data      | Code                          | Preserves test case metadata for Guardrails input | Loop Over Items                  | Check Guardrails                 | ## 📦 Preserve Original Metadata  Stores test case’s ID, expected result, category, description at top-level. |
| Check Guardrails            | Guardrails                    | Core safety evaluation applying multiple filters | Preserve Original Data           | Format Pass Result, Format Fail Result | ## 🛡️ Guardrails Evaluation  Applies safety filters; core benchmark.                                        |
| Format Pass Result          | Code                          | Formats PASS results into unified structure       | Check Guardrails                | Combine Result                  | ## 🧮 Normalize Result Format  Converts Guardrails output into consistent evaluation data.                    |
| Format Fail Result          | Code                          | Formats VIOLATION results into unified structure  | Check Guardrails                | Combine Result                  | ## 🧮 Normalize Result Format  Converts Guardrails output into consistent evaluation data.                    |
| Combine Result              | Merge                         | Merges PASS and VIOLATION results into one stream | Format Pass Result, Format Fail Result | Loop Over Items (done)           | ## 🔗 Merge Results  Collects PASS and VIOLATION outputs preserving sequence.                               |
| Calculate Metrics           | Code                          | Calculates accuracy, precision, recall, F1, etc. | Loop Over Items (done)           | Format Report                   | ## 📊 Build Accuracy Metrics  Generates confusion matrix, precision/recall/F1, category accuracy.            |
| Format Report              | Code                          | Builds detailed Markdown report from metrics      | Calculate Metrics               | Markdown                       | ## 📝 Build Markdown Report  Creates human-readable report with tables, metrics, recommendations.            |
| Markdown                   | Markdown                      | Converts Markdown report to HTML                   | Format Report                  | Send a message                 | ## 📄 Convert Markdown → HTML  Enables clean email formatting.                                               |
| Send a message             | Gmail                         | Sends final report email to recipient              | Markdown                      | None                          | ## ✉️ Email Report  Sends formatted Guardrails report; replace YOUR_MAIL_HERE with your email.               |
| Sticky Note                | Sticky Note                   | Workflow overview, instructions, and setup notes | None                          | None                          | ## 🛡️ Guardrails Evaluation Workflow  Explains purpose, usage, and setup.                                   |
| Sticky Note1               | Sticky Note                   | Explains test case generator rationale             | None                          | None                          | ## 📝 Test Case Generator  Consistent test suite avoids random results.                                      |
| Sticky Note2               | Sticky Note                   | Explains Loop Over Items node purpose               | None                          | None                          | ## 🔁 Loop Through Each Test Case  Clean per-item processing.                                                |
| Sticky Note3               | Sticky Note                   | Explains original metadata preservation             | None                          | None                          | ## 📦 Preserve Original Metadata  Keeps labels intact through Guardrails.                                   |
| Sticky Note4               | Sticky Note                   | Explains Guardrails evaluation core                  | None                          | None                          | ## 🛡️ Guardrails Evaluation  Core benchmark node.                                                           |
| Sticky Note5               | Sticky Note                   | Explains result normalization nodes                   | None                          | None                          | ## 🧮 Normalize Result Format  Consistent data for metrics calculation.                                     |
| Sticky Note6               | Sticky Note                   | Explains merging results                              | None                          | None                          | ## 🔗 Merge Results  Reunites pass and violation outputs correctly.                                         |
| Sticky Note7               | Sticky Note                   | Explains accuracy metric node                         | None                          | None                          | ## 📊 Build Accuracy Metrics  Generates key evaluation metrics.                                             |
| Sticky Note8               | Sticky Note                   | Explains Markdown report node                         | None                          | None                          | ## 📝 Build Markdown Report  Creates full test report including recommendations.                            |
| Sticky Note9               | Sticky Note                   | Explains Markdown to HTML conversion                   | None                          | None                          | ## 📄 Convert Markdown → HTML  For clean email formatting.                                                  |
| Sticky Note10              | Sticky Note                   | Explains email sending node                            | None                          | None                          | ## ✉️ Email Report  Sends report to your inbox; replace YOUR_MAIL_HERE.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: Manually start the workflow.

2. **Create Code node for test data**  
   - Name: "Set Test Data (code)"  
   - Paste JavaScript that returns the array of 36 test cases, each with: id, category, text, expected, description.  
   - Connect output of Manual Trigger to this node.

3. **Create SplitInBatches node**  
   - Name: "Loop Over Items"  
   - Configure batch size to 1 (default) to process each test case individually.  
   - Connect output of "Set Test Data (code)" to this node.

4. **Create Code node to preserve original metadata**  
   - Name: "Preserve Original Data"  
   - JavaScript: Extract top-level fields (`_id`, `_category`, `_expected`, `_description`) and include `text` field for Guardrails input.  
   - Connect "Loop Over Items" main output (batch output) to this node.

5. **Create Guardrails node**  
   - Name: "Check Guardrails"  
   - Configure Guardrails with the following safety categories and thresholds:  
     - PII: detect all types  
     - NSFW: threshold 0.7  
     - URLs: no allowed URLs, no subdomains allowed  
     - Keywords: "illegal, drugs, hack, exploit, weapon"  
     - Jailbreak: threshold 0.7  
     - Secret Keys: permissiveness balanced  
   - Use OpenAI API credentials for LLM backend.  
   - Connect output of "Preserve Original Data" to "Check Guardrails".

6. **Create Code node to format passing results**  
   - Name: "Format Pass Result"  
   - JavaScript: Match current item to preserved data by text, return unified JSON with actual "PASS", score 0, and other metadata fields.  
   - Connect "Check Guardrails" pass output (main output) to this node.

7. **Create Code node to format failing results**  
   - Name: "Format Fail Result"  
   - JavaScript: Match current item to preserved data by text, extract violation types and max confidence score, return unified JSON with actual "VIOLATION".  
   - Connect "Check Guardrails" fail output to this node.

8. **Create Merge node**  
   - Name: "Combine Result"  
   - Configure to merge outputs of "Format Pass Result" and "Format Fail Result", preserving order.  
   - Connect outputs of both format nodes to this node.

9. **Connect Merge node output to Loop Over Items "Done" output**  
   - This setup ensures that the merged results feed back to the batch done output for aggregation.

10. **Create Code node to calculate metrics**  
    - Name: "Calculate Metrics"  
    - JavaScript: Calculate confusion matrix, accuracy, precision, recall, F1 score, category-level accuracy, and prepare detailed results.  
    - Connect "Loop Over Items" done output to this node.

11. **Create Code node to format Markdown report**  
    - Name: "Format Report"  
    - JavaScript: Generate Markdown text report including overall metrics, confusion matrix, category performance, detailed results table, and actionable recommendations based on error counts.  
    - Connect output of "Calculate Metrics" to this node.

12. **Create Markdown node**  
    - Name: "Markdown"  
    - Configure to convert Markdown to HTML with emoji and tables enabled.  
    - Connect output of "Format Report" to this node.

13. **Create Gmail node to send email**  
    - Name: "Send a message"  
    - Configure recipient email address (replace "YOUR_MAIL_HERE" with actual email).  
    - Subject: "Guardrails Evaluation Report"  
    - Message body: Use HTML content from "Markdown" node.  
    - Provide Gmail OAuth2 credentials.  
    - Connect output of "Markdown" node to this node.

14. **Add Sticky Notes for clarity** (optional but recommended)  
    - Add workflow overview sticky note at top-left explaining purpose and usage.  
    - Add sticky notes near key nodes: test data generator, loop, preservation, Guardrails, formatting, merging, metrics, report, markdown conversion, and email sending.

15. **Test the workflow**  
    - Trigger manually via "When clicking ‘Execute workflow’".  
    - Verify email receipt and content correctness.  
    - Adjust test cases or thresholds as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow benchmarks n8n Guardrails node using 36 curated test cases across multiple safety categories. | Workflow overview sticky note within the workflow.                                                 |
| Replace placeholder `YOUR_MAIL_HERE` in Gmail node parameters with your real email address.          | Email sending configuration requirement.                                                           |
| Requires n8n version ≥ 1.119 and Guardrails node enabled.                                            | Workflow prerequisites as noted in sticky notes.                                                   |
| OpenAI credentials required for Guardrails node (uses GPT-4o-mini by default).                       | Credential setup prerequisite for Guardrails evaluation.                                           |
| Gmail OAuth2 credentials required for email delivery node.                                           | Credential setup prerequisite for email sending.                                                  |
| The test suite includes edge cases to test borderline and obfuscated content (e.g., obfuscated PII). | Test data includes both direct and edge case examples to stress test Guardrails.                   |
| Report includes actionable recommendations based on false positives and negatives detected.          | Useful for tuning guardrail thresholds and test coverage.                                          |
| Markdown report includes tables, emoji, and detailed category breakdowns, converted to HTML for email.| Improves readability and presentation for recipients.                                             |
| You can customize test cases in "Set Test Data (code)" and adjust guardrail thresholds in "Check Guardrails". | Enables workflow tailoring for specific safety policies or different LLMs.                         |
| For alternative notification channels, replace Gmail node with Slack, Teams, or HTTP request nodes.  | Extensibility note for integration with other communication platforms.                             |
| Guardrails node configuration uses a balanced approach in secret keys permissiveness and threshold tuning for NSFW and jailbreak detection. | Balances false positives vs. false negatives in safety detection.                                  |

---

This document fully describes the "Benchmark Content Safety Guardrails with Automated Test Suite & Reports" workflow, enabling advanced users and automation agents to understand, reproduce, and customize the workflow with confidence.