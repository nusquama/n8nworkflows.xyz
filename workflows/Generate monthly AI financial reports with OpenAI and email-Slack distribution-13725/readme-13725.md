Generate monthly AI financial reports with OpenAI and email/Slack distribution

https://n8nworkflows.xyz/workflows/generate-monthly-ai-financial-reports-with-openai-and-email-slack-distribution-13725


# Generate monthly AI financial reports with OpenAI and email/Slack distribution

# AI Financial Report Analyzer

This n8n workflow automates the end-to-end process of financial data analysis. It fetches data from accounting systems, calculates key performance indicators (KPIs), detects anomalies, uses OpenAI to generate executive-level insights, and distributes a professional HTML report via Email and Slack.

---

### 1. Workflow Overview

The workflow is structured into five functional phases:

*   **1.1 Data Extraction:** Triggered monthly to pull P&L, Balance Sheet, and Cash Flow data from external APIs.
*   **1.2 Normalization & Calculation:** Standardizes raw JSON data into financial metrics (Margins, Ratios, Working Capital) and calculates a "Financial Health Score."
*   **1.3 Trend & Anomaly Detection:** Applies business logic to identify critical risks (e.g., liquidity crises) and strengths.
*   **1.4 AI Synthesis & Reporting:** Sends processed data to OpenAI (GPT-4o-mini) for natural language interpretation and generates a styled HTML report.
*   **1.5 Distribution & Archiving:** Stores the record in a PostgreSQL database and alerts stakeholders via Slack and Email.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Extract Data
This block initiates the process and gathers all necessary financial inputs.
*   **Nodes Involved:** `Monthly analysis on 1st at 8 AM`, `Fetch P&L statement`, `Fetch balance sheet`, `Fetch cash flow statement`, `Fetch previous period data for comparison`.
*   **Node Details:**
    *   **Schedule Trigger:** Configured with a Cron expression (`0 8 1 * *`) to run at 08:00 on the first day of every month.
    *   **HTTP Requests:** Four nodes performing `GET` requests to a placeholder accounting API (`api.accounting.example.com`). These must be replaced with actual endpoints (QuickBooks, Xero, etc.).
    *   **Failure Modes:** API timeouts, authentication expiration (OAuth2), or schema changes in the accounting software.

#### 2.2 Data Normalization & Validation
Combines disparate API responses into a single object and calculates financial formulas.
*   **Nodes Involved:** `Merge all financial statements`, `Normalize and validate financial data`.
*   **Node Details:**
    *   **Merge Node:** Uses "Combine" mode to aggregate the four HTTP inputs into one dataset.
    *   **Code Node (JavaScript):** Calculates ~20+ metrics including Gross/Net Margin, EBITDA, Current Ratio, Debt-to-Equity, and Cash Conversion Cycle. It assigns a Health Score (0-100) based on weighted thresholds.
    *   **Edge Cases:** Division by zero (handled by checks for `revenue > 0`), missing fields (defaulted to `0`), and null values.

#### 2.3 Analysis & Anomaly Detection
Evaluates the calculated metrics against business rules to flag issues.
*   **Nodes Involved:** `Analyze trends and detect anomalies`.
*   **Node Details:**
    *   **Logic:** Categorizes trends (Growing vs. Decline) and pushes strings into `critical_anomalies` or `warnings` arrays. 
    *   **Thresholds:** Flags critical if Current Ratio < 1.0, Net Margin < 0, or Debt-to-Equity > 2.0.
    *   **Output:** A JSON object enriched with `strategic_recommendations` and a `performance_rating`.

#### 2.4 AI Insights & Report Generation
Transforms technical data into human-readable executive summaries.
*   **Nodes Involved:** `Generate AI-powered insights`, `Extract and format AI insights`, `Generate professional HTML report`.
*   **Node Details:**
    *   **OpenAI Node (HTTP):** Uses `gpt-4o-mini`. System prompt defines the role as "Financial Analyst." User prompt sends current metrics and anomalies for synthesis.
    *   **HTML Generation:** A Code node that wraps all metrics and AI insights into a self-contained, CSS-styled HTML template with responsive tables and color-coded badges (Green for Excellent, Red for Critical).

#### 2.5 Distribution & Notification
Finalizes the workflow by archiving and communicating results.
*   **Nodes Involved:** `Store report in database`, `Post report summary to Slack`, `Email report to management`, `Log report generation completion`.
*   **Node Details:**
    *   **Postgres:** Inserts the report data into a `financial_reports` table.
    *   **Slack:** Sends a high-level summary (Revenue, Net Income, Health Score) using Slack Blocks for formatting.
    *   **Email (SMTP):** Attaches the full HTML report to the body of the email.
    *   **Completion Log:** Updates n8n's Static Data to track cumulative statistics (Total reports, average health score over time).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Monthly analysis on 1st... | Schedule Trigger | Workflow Entry | None | Fetch P&L, Balance, etc. | 1. Trigger & Extract Data |
| Fetch P&L statement | HTTP Request | Data Extraction | Schedule Trigger | Merge | 1. Trigger & Extract Data |
| Fetch balance sheet | HTTP Request | Data Extraction | Schedule Trigger | Merge | 1. Trigger & Extract Data |
| Merge all financial... | Merge | Data Aggregation | HTTP Request nodes | Normalize | 2. Data Normalization... |
| Normalize and validate... | Code | KPI Calculation | Merge | Analyze trends | 2. Data Normalization... |
| Analyze trends... | Code | Logic Engine | Normalize | Generate AI insights | 3. Analysis & Anomaly... |
| Generate AI-powered... | HTTP Request | AI Synthesis | Analyze trends | Extract AI insights | 4. AI Insights & Report... |
| Extract and format AI... | Code | Formatting | Generate AI insights | Generate HTML | 4. AI Insights & Report... |
| Generate professional HTML | Code | Report Rendering | Extract AI insights | Store in database | 4. AI Insights & Report... |
| Store report in database | Postgres | Persistence | Generate HTML | Slack, Email | 5. Distribution &... |
| Post report summary... | HTTP Request | Notification | Store in database | Log completion | 5. Distribution &... |
| Email report to... | Email Send | Distribution | Store in database | Log completion | 5. Distribution &... |
| Log report generation... | Code | Analytics | Slack, Email | None | 5. Distribution &... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger:** Add a **Schedule Trigger** set to the 1st of every month at 08:00.
2.  **API Extraction:**
    *   Add 4 **HTTP Request** nodes.
    *   Configure them to point to your accounting software (e.g., QuickBooks or Xero).
    *   Ensure they fetch: Income Statement, Balance Sheet, Cash Flow, and Historical (Previous Period) data.
3.  **Merge:** Connect all 4 HTTP nodes to a **Merge** node. Set Mode to `Combine` and "Wait for all inputs."
4.  **Logic (Normalization):** 
    *   Add a **Code** node. 
    *   Implement calculations for: `Gross Margin`, `Operating Margin`, `Current Ratio`, `Debt-to-Equity`, and `Return on Equity`.
    *   Create a "Health Score" logic based on these ratios.
5.  **Logic (Anomalies):** 
    *   Add another **Code** node. 
    *   Use `if/else` statements to check for risks (e.g., `if (current_ratio < 1) push "Liquidity Crisis"`).
6.  **AI Integration:**
    *   Add an **HTTP Request** node for OpenAI.
    *   Method: `POST`, URL: `https://api.openai.com/v1/chat/completions`.
    *   Payload: Include a system message ("Financial Analyst") and a user message containing the metrics from the previous node.
7.  **Report Formatting:**
    *   Add a **Code** node to build a template string. Use HTML/CSS to map the JSON variables into a professional layout.
8.  **Storage:** Add a **Postgres** node (or Google Sheets/Airtable). Map the report fields to your database columns.
9.  **Notifications:**
    *   Add an **HTTP Request** node for Slack Webhooks. Use a JSON body with `blocks` for formatting.
    *   Add an **Email Send** node. Map the `report_html` property to the Body field.
10. **Final Log:** Add a **Code** node to use `$getWorkflowStaticData` to increment report counts and log the `report_id` to the n8n console.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Step: Accounting APIs** | Connect QuickBooks, Xero, SAP, or financial database. |
| **Setup Step: AI/LLM** | Requires OpenAI (GPT-4) or Claude API key. |
| **Setup Step: PDF/HTML** | HTML is generated via Code; can be converted to PDF using a 'Convert to File' or 'Chrome' node. |
| **Project Purpose** | Automates executive summaries and anomaly detection for management. |