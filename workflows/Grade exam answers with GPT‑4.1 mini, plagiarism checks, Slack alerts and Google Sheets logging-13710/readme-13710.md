Grade exam answers with GPT‑4.1 mini, plagiarism checks, Slack alerts and Google Sheets logging

https://n8nworkflows.xyz/workflows/grade-exam-answers-with-gpt-4-1-mini--plagiarism-checks--slack-alerts-and-google-sheets-logging-13710


# Grade exam answers with GPT‑4.1 mini, plagiarism checks, Slack alerts and Google Sheets logging

This technical document provides a comprehensive analysis of the "Smart examination marking with plagiarism detection and moderation" n8n workflow. This system utilizes a multi-agent AI architecture to automate the grading process while maintaining academic integrity and quality control.

---

### 1. Workflow Overview

The workflow automates the end-to-end process of grading student examinations. It retrieves student submissions and rubrics, interprets the marking criteria, performs a primary assessment, checks for plagiarism, moderates the results, and generates student feedback. The logic is divided into five functional blocks:

*   **1.1 Data Initialization & Interpretation:** Configuration of environment variables and structured parsing of the grading rubric.
*   **1.2 Primary Assessment:** The initial AI grading of the student answer based on the interpreted rubric.
*   **1.3 Integrity Verification:** Parallel analysis of potential plagiarism or academic misconduct if flagged during primary marking.
*   **1.4 Quality Moderation & Routing:** Review of the primary grade with logic to approve, adjust (secondary marking), or escalate (manual review).
*   **1.5 Reporting & Logging:** Generation of statistics, Slack alerts for escalations, and archival of results in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Initialization & Interpretation
**Overview:** Sets the operational parameters (passing grades, thresholds) and retrieves the raw data. An AI agent then converts the raw rubric into a structured JSON schema for consistent grading.
*   **Nodes Involved:** `Manual Trigger`, `Workflow Configuration`, `Retrieve Student Answer and Rubric`, `Rubric Interpreter Agent`, `OpenAI Chat Model - Rubric`, `Structured Output Parser - Rubric`.
*   **Node Details:**
    *   **Workflow Configuration (Set):** Defines placeholders for `apiEndpoint`, `totalMarks` (100), `passingGrade` (50), `slackChannel`, `googleSheetId`, and `plagiarismThreshold` (0.7).
    *   **Retrieve Student Answer and Rubric (HTTP Request):** Performs a GET request to the configured API endpoint. Failure to reach the URL will stop the workflow.
    *   **Rubric Interpreter Agent (AI Agent):** Uses `gpt-4.1-mini`. It parses weightings, grade levels (excellent to poor), and key concepts. Output is constrained by a JSON schema via the `Structured Output Parser`.

#### 1.2 Primary Assessment
**Overview:** This block performs the actual grading. The AI compares the student's answer against the structured rubric criteria.
*   **Nodes Involved:** `Primary Marker Agent`, `OpenAI Chat Model - Marker`, `Structured Output Parser - Marker`.
*   **Node Details:**
    *   **Primary Marker Agent:** Analyzes the student's text. It is configured to provide justifications for every mark awarded and identify specific strengths and weaknesses.
    *   **Key Expressions:** References `{{ $('Retrieve Student Answer and Rubric').first().json.studentAnswer }}`.
    *   **Edge Cases:** If the student answer is missing or in an unsupported format, the agent may return a low confidence score.

#### 1.3 Integrity Verification
**Overview:** Initiated only if the Primary Marker identifies suspicious patterns. It performs a deep dive into text similarity and phrasing inconsistencies.
*   **Nodes Involved:** `Check Integrity Flags`, `Plagiarism Analyzer Agent`, `OpenAI Chat Model - Plagiarism`, `Structured Output Parser - Plagiarism`, `Prepare Integrity Report`.
*   **Node Details:**
    *   **Check Integrity Flags (If):** Evaluates if the `integrityFlags` array length is greater than 0.
    *   **Plagiarism Analyzer Agent:** Assesses severity (LOW to CRITICAL) and recommends actions (INVESTIGATE/REJECT).
    *   **Edge Cases:** False positives in common technical terms; requires the `plagiarismThreshold` defined in Block 1.1.

#### 1.4 Quality Moderation & Routing
**Overview:** Acts as the "Senior Examiner." It reviews the primary marker's output for bias or inconsistency and decides the next step.
*   **Nodes Involved:** `Quality Moderator Agent`, `Feedback Generator Agent`, `Route by Moderation Decision`, `Secondary Marker Agent`.
*   **Node Details:**
    *   **Quality Moderator Agent:** Reviews the alignment between evidence provided and marks awarded. Can change the decision to `APPROVED`, `ADJUSTED`, or `ESCALATE`.
    *   **Route by Moderation Decision (Switch):** Directs the workflow based on the Moderator's string output.
    *   **Secondary Marker Agent:** Triggered only if the status is `ADJUSTED`. It performs a blind second marking to resolve discrepancies.

#### 1.5 Reporting & Logging
**Overview:** Finalizes the process by calculating cohort statistics, notifying staff of issues, and logging data.
*   **Nodes Involved:** `Prepare Escalation Data`, `Send Escalation Alert`, `Calculate Statistics`, `Merge All Paths`, `Final Output Compilation`, `Log to Google Sheets`.
*   **Node Details:**
    *   **Send Escalation Alert (Slack):** Sends a formatted block to Slack including Student ID and reason for escalation.
    *   **Calculate Statistics (Code):** A JavaScript node that calculates Mean, Median, Standard Deviation, Pass/Fail rates, and Grade Distribution for the current batch.
    *   **Log to Google Sheets:** Appends the final object (including breakdown and feedback) to a specified spreadsheet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manual Trigger | manualTrigger | Workflow Entry | None | Workflow Configuration | |
| Workflow Configuration | set | Variable Definition | Manual Trigger | Retrieve Student Answer... | |
| Retrieve Student Answer... | httpRequest | Data Fetching | Workflow Configuration | Rubric Interpreter Agent | Trigger, Data Retrieval & Rubric Interpretation: Fetches student answers and rubrics. |
| Rubric Interpreter Agent | agent | Logic Structuring | Retrieve Student Answer... | Primary Marker Agent | Ensures marking begins with complete, validated inputs. |
| Primary Marker Agent | agent | Core Assessment | Rubric Interpreter Agent | Moderator, Integrity IF, Merge | Delivers objective, criteria-aligned scoring at scale. |
| Quality Moderator Agent | agent | Quality Control | Primary Marker, Merge Integrity, Merge Marking | Feedback Generator Agent | Reviews marking quality and routes borderline cases for secondary marking. |
| Feedback Generator Agent | agent | Student Comms | Quality Moderator Agent | Route by Moderation Decision | Closes the assessment loop with actionable feedback and full audit records. |
| Route by Moderation Decision| switch | Logic Routing | Feedback Generator Agent | Secondary Marker, Escalation, Statistics | Define moderation thresholds in the Route by Moderation Decision rules node. |
| Check Integrity Flags | if | Conditional Gate | Primary Marker Agent | Plagiarism Agent (True), Moderator (False) | Checks submissions for integrity flags in parallel with marking. |
| Plagiarism Analyzer Agent | agent | Fraud Detection | Check Integrity Flags | Prepare Integrity Report | Embeds academic integrity verification without slowing the marking pipeline. |
| Secondary Marker Agent | agent | Conflict Resolution | Route by Moderation Decision| Merge Marking Results | |
| Calculate Statistics | code | Analytics | Route by Moderation Decision| Merge All Paths | |
| Send Escalation Alert | slack | Notification | Prepare Escalation Data | Merge All Paths | Configure Slack credentials and set escalation alert channel. |
| Log to Google Sheets | googleSheets | Archiving | Final Output Compilation | None | Google Sheets with service account credentials. |
| Final Output Compilation | set | Data Normalization| Merge All Paths | Log to Google Sheets | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Create a new n8n workflow and place a **Manual Trigger**.
2.  **Configuration:** Add a **Set** node ("Workflow Configuration"). Create string parameters for your API URL, Slack Channel ID, and Google Sheet ID. Set numeric parameters for `totalMarks` and `passingGrade`.
3.  **Data Ingestion:** Add an **HTTP Request** node. Set the URL to use the expression from the Configuration node.
4.  **AI Infrastructure (Rubric):** 
    *   Add an **AI Agent** node ("Rubric Interpreter"). 
    *   Connect an **OpenAI Chat Model** (Model: `gpt-4.1-mini`).
    *   Connect a **Structured Output Parser**. Define a JSON schema including `totalMarksAvailable` (number) and `criteria` (array of objects).
5.  **AI Infrastructure (Primary Marking):** 
    *   Duplicate the Agent setup for "Primary Marker".
    *   Update the System Message to focus on grading and evidence citation.
    *   Define a JSON schema in the parser to include `marksBreakdown`, `integrityFlags`, and `confidenceLevel`.
6.  **Integrity Logic:**
    *   Add an **If** node. Check if `integrityFlags` length > 0.
    *   If True, add another **AI Agent** ("Plagiarism Analyzer") with a specialized prompt for academic misconduct.
7.  **Moderation Logic:**
    *   Add the "Quality Moderator" **AI Agent**. Connect its input to the Primary Marker and the Integrity Report.
    *   Add a **Switch** node to route outputs based on the Moderator's decision: `APPROVED`, `ADJUSTED`, or `ESCALATE`.
8.  **Feedback & Secondary Marking:**
    *   Add an AI Agent ("Feedback Generator") before the Switch node to ensure every student gets feedback regardless of routing.
    *   Connect the `ADJUSTED` path to a "Secondary Marker" AI Agent.
9.  **External Integrations:**
    *   Connect the `ESCALATE` path to a **Set** node (formatting Slack text) and then a **Slack** node.
    *   Connect the `APPROVED` path to a **Code** node. Use JavaScript to map through `$input.all()` and calculate cohorts' average and standard deviation.
10.  **Final Aggregation:** Use a **Merge** node (Mode: Choose "Wait for All Inputs" or "Combine" depending on your batching strategy) to collect data from the Slack path, Statistics path, and Secondary Marker.
11.  **Logging:** Add a final **Set** node to clean up the object, then a **Google Sheets** node using the "Append" operation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | Requires OpenAI API Key, Slack OAuth2, and Google Sheets Service Account/OAuth2. |
| **Model Selection** | While `gpt-4.1-mini` is used here for cost-efficiency, `gpt-4o` is recommended for complex legal or medical rubrics. |
| **Scaling** | For high-volume exams, ensure n8n is configured to handle large executions in a queue (Redis/RabbitMQ). |
| **Customization** | The "Code" node can be modified to push results to an LMS like Moodle or Canvas via their respective APIs. |