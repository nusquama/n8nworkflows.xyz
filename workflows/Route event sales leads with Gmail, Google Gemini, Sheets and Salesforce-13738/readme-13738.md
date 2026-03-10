Route event sales leads with Gmail, Google Gemini, Sheets and Salesforce

https://n8nworkflows.xyz/workflows/route-event-sales-leads-with-gmail--google-gemini--sheets-and-salesforce-13738


# Route event sales leads with Gmail, Google Gemini, Sheets and Salesforce

This technical reference document provides a detailed breakdown of the **Email Sentiment Router for Event Sales Leads** workflow. This system automates the intake, classification, and CRM integration of inbound event-related inquiries using Google Gemini AI, Gmail, Salesforce, and Google Sheets.

---

### 1. Workflow Overview

The workflow is designed to manage high-volume event inquiries (sponsorship, exhibitions, attendance) by using Generative AI to categorize sentiment and extract structured business intelligence. It ensures that "Hot Leads" are prioritized for immediate sales outreach while maintaining a comprehensive analytics trail for marketing and management.

#### Logical Blocks:
*   **1.1 Input Reception:** Monitors a Gmail inbox for new emails or triggers an evaluation batch from an internal data table.
*   **1.2 AI Analysis:** A multi-step AI process that determines sentiment (Positive, Neutral, Negative) and extracts structured data (Topic, Intent, Urgency, Budget).
*   **1.3 Analytics & Logging:** Transforms raw data into a clean format and logs it simultaneously to n8n Data Tables and Google Sheets.
*   **1.4 CRM Integration:** Upserts leads into Salesforce, creates follow-up tasks, and conditionally generates opportunities for high-value prospects.
*   **1.5 Routing & Notifications:** Sends context-aware notifications via Slack and Gmail based on the calculated sentiment.
*   **1.6 Evaluation Framework:** A dedicated branch for testing AI accuracy against a "golden" dataset without triggering production notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** Captures incoming communications. It supports both live production (Gmail) and testing (Evaluation Trigger).
*   **Nodes Involved:** `Monitor Gmail Inbox`, `Fetch Evaluation Dataset Row`, `Loop Evaluation Batch`.
*   **Node Details:**
    *   **Monitor Gmail Inbox:** (Gmail Trigger v1.2) Polls for new messages. Outputs full email metadata (Body, Sender, Subject).
    *   **Fetch Evaluation Dataset Row:** (Evaluation Trigger v4.7) Initiates a test run using a predefined data table (`sWTqB6DnzaXG6be6`).
    *   **Loop Evaluation Batch:** (Split in Batches v3) Iterates through the test dataset for bulk AI evaluation.

#### 2.2 AI Analysis
*   **Overview:** Employs Google Gemini to transform unstructured text into structured intelligence.
*   **Nodes Involved:** `Analyze Email Sentiment`, `Google Gemini 2.5 Flash Lite`, `Extract Email Intelligence`, `Google Gemini for Enrichment`.
*   **Node Details:**
    *   **Analyze Email Sentiment:** (Sentiment Analysis v1.1) Uses Gemini 2.0 Flash to categorize the email as Positive, Neutral, or Negative. Includes a specialized system prompt for the events industry focusing on buying intent.
    *   **Extract Email Intelligence:** (Information Extractor v1) Extracts 9 specific attributes including `primary_topic`, `urgency_score` (1-10), `budget_mentioned` (boolean), and `suggested_action`.
    *   **Edge Cases:** Failure to extract structured JSON is mitigated by `maxTries: 2` and `retryOnFail`.

#### 2.3 Analytics & Logging
*   **Overview:** Prepares a flat data structure for reporting and logs it to external sources.
*   **Nodes Involved:** `Prepare Analytics Row`, `Log Email Analytics`, `Append to Google Sheets`.
*   **Node Details:**
    *   **Prepare Analytics Row:** (Code v2) JavaScript logic that cleans email addresses, extracts domains, and calculates the ISO week number for reporting.
    *   **Append to Google Sheets:** (Google Sheets v4.5) Adds a row to a dashboard spreadsheet. Uses `continueOnFail` to ensure CRM sync isn't blocked by Google API issues.

#### 2.4 CRM Integration (Salesforce)
*   **Overview:** Syncs lead data to Salesforce and manages the sales pipeline.
*   **Nodes Involved:** `Prepare CRM Data`, `Upsert Lead to Salesforce`, `Check Create Opportunity`, `Create Opportunity`, `Create Follow-up Task`.
*   **Node Details:**
    *   **Prepare CRM Data:** (Code v2) Logic to determine if an Opportunity should be created. Criteria: `(Sentiment == Positive AND Urgency >= 7) OR Budget Mentioned == true`.
    *   **Upsert Lead to Salesforce:** (Salesforce v1) Uses the email address as the External ID to prevent duplicate records. Maps AI-extracted fields to custom Salesforce fields (e.g., `Urgency_Score__c`).
    *   **Create Follow-up Task:** (Salesforce v1) Automatically sets a task for the sales owner due in 2 days, containing the AI's `suggestedAction`.

#### 2.5 Routing & Notifications
*   **Overview:** Delivers the right information to the right channel based on sentiment.
*   **Nodes Involved:** `Route by Sentiment`, `Gate [Positive/Neutral/Negative] Route`, `Send [Hot Lead/Follow-up/Insights] Email`, `Post Hot Lead to Slack`.
*   **Node Details:**
    *   **Gate Nodes:** (Evaluation v4.8) Uses the `checkIfEvaluating` operation. If the workflow was started by the Evaluation Trigger, these nodes block the output to prevent sending real emails/Slack messages during tests.
    *   **Gmail Nodes:** (Gmail v2.1) Sends formatted HTML emails with original message excerpts and recommended actions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Monitor Gmail Inbox | gmailTrigger | Trigger | (None) | Analyze Email Sentiment | 1. Email Trigger: Monitors Gmail inbox every minute. |
| Analyze Email Sentiment | sentimentAnalysis | AI Processing | Monitor Gmail / Loop Eval | Route by Sentiment, Extract Intelligence | 2. AI Sentiment & Intelligence Extraction: Step 1. |
| Extract Email Intelligence | informationExtractor | AI Processing | Analyze Email Sentiment | Prepare Analytics Row | 2. AI Sentiment & Intelligence Extraction: Step 2. |
| Route by Sentiment | switch | Logic/Routing | Analyze Email Sentiment | Gate Routes | 2. AI Sentiment & Intelligence Extraction: Step 3. |
| Gate Positive Route | evaluation | Safety Gate | Route by Sentiment | Save Eval / Gmail / Slack | 3. Sentiment Routing Gate: Routes emails to correct team. |
| Gate Neutral Route | evaluation | Safety Gate | Route by Sentiment | Save Eval / Gmail | 3. Sentiment Routing Gate: Routes emails to correct team. |
| Gate Negative Route | evaluation | Safety Gate | Route by Sentiment | Save Eval / Gmail | 3. Sentiment Routing Gate: Routes emails to correct team. |
| Prepare Analytics Row | code | Data Prep | Extract Intelligence | Log Analytics / Sheets / CRM | 4. Analytics Logging: Logs every processed email. |
| Log Email Analytics | dataTable | Storage | Prepare Analytics Row | (None) | 4. Analytics Logging: Logs to n8n Data Table. |
| Append to Google Sheets | googleSheets | External Storage | Prepare Analytics Row | (None) | 4. Analytics Logging: Logs to Google Sheets for Looker. |
| Prepare CRM Data | code | Logic | Prepare Analytics Row | Upsert Salesforce | 3. CRM Integration: Every email syncs to Salesforce. |
| Upsert Lead to Salesforce | salesforce | CRM | Prepare CRM Data | Check Opportunity / Create Task | 3. CRM Integration: Upsert Lead (deduped). |
| Create Opportunity | salesforce | CRM | Check Create Opportunity | (None) | 3. CRM Integration: Only for hot leads. |
| Post Hot Lead to Slack | slack | Notification | Gate Positive Route | (None) | 5. Team Notifications: Positive -> #hot-leads. |
| Record Evaluation Metrics | evaluation | Evaluation | Save Evaluation Output | Loop Evaluation Batch | Records AI output vs. expected labels. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Add a **Gmail Trigger** node. Set poll frequency (e.g., every 1-5 minutes).
    *   (Optional) Add an **Evaluation Trigger** and a **Split in Batches** node to handle test datasets.
2.  **AI Layer Configuration:**
    *   Add **Analyze Email Sentiment**. Connect a **Google Gemini Chat Model** (Gemini 2.0 Flash suggested). Configure the System Prompt to define "Positive", "Neutral", and "Negative" within your specific industry context.
    *   Add **Extract Email Intelligence**. Define attributes: `primary_topic`, `intent`, `urgency_score` (Number), `org_name`, `budget_mentioned` (Boolean), and `suggested_action`.
3.  **Data Transformation:**
    *   Add a **Code Node** to consolidate data from the AI nodes and the Gmail trigger. Ensure you extract the sender's email address and domain.
4.  **CRM & Storage Integration:**
    *   **Salesforce:** Create an **Upsert Lead** node. Use `Email` as the matching field. Map AI variables to description and custom fields.
    *   **Salesforce Logic:** Add an **If Node** to check if `urgency_score >= 7`. If true, connect to a **Salesforce Opportunity** node.
    *   **Google Sheets:** Create an **Append** node to log the cleaned data for BI tools.
5.  **Routing Logic:**
    *   Add a **Switch Node** based on `sentimentAnalysis.category`. Create three outputs: Positive, Neutral, Negative.
    *   Add **Evaluation (Check if Evaluating)** nodes for each branch to prevent testing data from reaching customers/staff.
6.  **Notification Setup:**
    *   **Slack:** Add nodes to post to `#hot-leads`, `#follow-ups`, or `#insights`.
    *   **Gmail:** Add nodes to email the sales team with the lead details and AI-suggested next steps.
7.  **Credentials:**
    *   Configure OAuth2 for Gmail, Slack, Google Sheets, and Salesforce.
    *   Configure API Key for Google Gemini.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Feedback & Consulting** | [Start a conversation with Braia Labs](https://tally.so/r/EkKGgB) |
| **Professional Profile** | [Milo Bravo - LinkedIn](https://linkedin.com/in/MiloBravo/) |
| **System Architecture** | Built by Braia Labs for Automation & BI Integration. |
| **Error Handling** | CRM and Logging nodes utilize `continueOnFail` to prioritize lead routing over secondary data logging. |