Route and qualify email leads with Gmail, Gemini, Slack, Sheets and Salesforce

https://n8nworkflows.xyz/workflows/route-and-qualify-email-leads-with-gmail--gemini--slack--sheets-and-salesforce-13740


# Route and qualify email leads with Gmail, Gemini, Slack, Sheets and Salesforce

This document provides a comprehensive technical reference for the **Email Sentiment Router** workflow. This system automates the intake, qualification, and CRM logging of event-related sales inquiries using Google Gemini AI and Salesforce.

---

### 1. Workflow Overview

The workflow is designed to manage high volumes of inbound emails for event sales teams. It transforms raw email text into structured business intelligence, ensuring that high-value leads are fast-tracked to the sales pipeline while maintaining a comprehensive analytics dashboard.

**Logical Blocks:**
*   **1.1 Input Reception:** Monitors a Gmail inbox or an evaluation dataset for incoming messages.
*   **1.2 AI Intelligence Extraction:** Two-stage AI processing to determine sentiment and extract structured entities (budget, urgency, topic).
*   **1.3 Data Synchronization:** Parallel logging to n8n Data Tables and Google Sheets for BI reporting.
*   **1.4 CRM Integration:** Salesforce automation that upserts leads, creates follow-up tasks, and conditionally generates opportunities.
*   **1.5 Multi-Channel Routing:** Logic-based routing that notifies the team via Slack and Email based on the lead's sentiment.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** Captures the source data. It supports both live production (Gmail) and a testing framework (Evaluation Dataset).
*   **Nodes Involved:** `Monitor Gmail Inbox`, `Fetch Evaluation Dataset Row`, `Loop Evaluation Batch`.
*   **Node Details:**
    *   **Monitor Gmail Inbox:** (Gmail Trigger) Polls the inbox. Configured to capture full email body and metadata.
    *   **Fetch Evaluation Dataset Row:** (Evaluation Trigger) Used for testing AI accuracy against a "golden" dataset.

#### 2.2 AI Intelligence Extraction
*   **Overview:** Uses LLMs to interpret the "unstructured" email content into "structured" data.
*   **Nodes Involved:** `Analyze Email Sentiment`, `Google Gemini 2.5 Flash Lite`, `Extract Email Intelligence`, `Google Gemini for Enrichment`.
*   **Node Details:**
    *   **Analyze Email Sentiment:** Uses a LangChain Sentiment Analysis node. Categorizes as *Positive, Neutral, or Negative*. Includes a custom system prompt focused on event sponsorship/exhibition.
    *   **Extract Email Intelligence:** Uses LangChain Information Extractor. It identifies `urgency_score` (1-10), `budget_mentioned` (boolean), `primary_topic`, and `org_name`.
    *   **Potential Failures:** LLM timeouts or API rate limits. The workflow uses `maxTries: 2` and `retryOnFail` to mitigate this.

#### 2.3 Analytics & Data Preparation
*   **Overview:** Processes the raw AI output into a format suitable for databases and CRM.
*   **Nodes Involved:** `Prepare Analytics Row`, `Log Email Analytics`, `Append to Google Sheets`.
*   **Node Details:**
    *   **Prepare Analytics Row:** (Code Node) A JavaScript block that cleans the email sender string, calculates the week number, and maps AI outputs to schema-ready JSON.
    *   **Log Email Analytics:** (Data Table) Writes to an internal n8n table for quick auditing.
    *   **Append to Google Sheets:** (Google Sheets) Sends data to a spreadsheet, typically used as a data source for Looker Studio.

#### 2.4 CRM Integration (Salesforce)
*   **Overview:** Updates the sales pipeline without manual data entry.
*   **Nodes Involved:** `Prepare CRM Data`, `Upsert Lead to Salesforce`, `Check Create Opportunity`, `Create Opportunity`, `Create Follow-up Task`.
*   **Node Details:**
    *   **Upsert Lead to Salesforce:** Uses `email` as the External ID to prevent duplicates. Maps custom fields like `Sentiment__c` and `Urgency_Score__c`.
    *   **Check Create Opportunity:** (IF Node) Filters for `createOpportunity == true`.
    *   **Create Opportunity:** Triggered only if Sentiment is *Positive* AND Urgency is ≥ 7.
    *   **Create Follow-up Task:** Generates a task for the lead owner with an AI-suggested action, due in 2 days.

#### 2.5 Multi-Channel Routing
*   **Overview:** Notifies human agents based on the priority of the lead.
*   **Nodes Involved:** `Route by Sentiment`, `Gate Nodes` (Positive/Neutral/Negative), `Send Email` (x3), `Post Hot Lead to Slack`.
*   **Node Details:**
    *   **Gate Nodes:** (Evaluation Node) Checks `checkIfEvaluating`. This ensures that if the workflow is in "Test Mode," it doesn't send real emails to the sales team.
    *   **Post Hot Lead to Slack:** Sends a formatted block to `#hot-leads` including a snippet of the email and the recommended action.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Monitor Gmail Inbox | Gmail Trigger | Production Input | - | Analyze Email Sentiment | 1. Email Trigger |
| Analyze Email Sentiment | Sentiment Analysis | AI Classification | Gmail Inbox / Loop Batch | Route by Sentiment, Extract Email Intelligence | 2. AI Sentiment & Intelligence Extraction |
| Extract Email Intelligence | Information Extractor | AI Data Extraction | Analyze Email Sentiment | Prepare Analytics Row | 2. AI Sentiment & Intelligence Extraction |
| Route by Sentiment | Switch | Logical Routing | Analyze Email Sentiment | Gate Nodes (x3) | 3. Sentiment Routing Gate |
| Prepare Analytics Row | Code | Data Formatting | Extract Email Intelligence | Log Analytics, Append Sheets, Prepare CRM | 4. Analytics Logging |
| Log Email Analytics | Data Table | Internal Storage | Prepare Analytics Row | - | 4. Analytics Logging |
| Append to Google Sheets | Google Sheets | External Dashboarding | Prepare Analytics Row | - | 4. Analytics Logging |
| Prepare CRM Data | Code | CRM Transformation | Prepare Analytics Row | Upsert Lead to Salesforce | 6. CRM Integration (Salesforce) |
| Upsert Lead to Salesforce | Salesforce | Lead Sync | Prepare CRM Data | Check Opp, Create Task | 6. CRM Integration (Salesforce) |
| Check Create Opportunity | If | Qualification Filter | Upsert Lead | Create Opportunity | 6. CRM Integration (Salesforce) |
| Create Opportunity | Salesforce | High-Value Automation | Check Create Opp | - | 6. CRM Integration (Salesforce) |
| Create Follow-up Task | Salesforce | Task Automation | Upsert Lead | - | 6. CRM Integration (Salesforce) |
| Post Hot Lead to Slack | Slack | Immediate Alert | Gate Positive Route | - | 5. Team Notifications |
| Send Hot Lead Email | Gmail | Priority Alert | Gate Positive Route | - | 5. Team Notifications |
| Fetch Evaluation Dataset | Evaluation Trigger | Testing Input | - | Loop Evaluation Batch | Evaluation Framework (Optional) |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Add a **Gmail Trigger**. Configure it to `everyMinute` (or as needed) to watch for new emails.
    *   (Optional) Add an **Evaluation Trigger** for testing with a CSV/Data Table.

2.  **AI Configuration:**
    *   Add a **Sentiment Analysis** node. Connect a **Google Gemini Chat** model. Provide a system prompt defining what constitutes "Positive" (e.g., interest in sponsorship) vs "Negative" (cancellation).
    *   Add an **Information Extractor** node. Define attributes: `urgency_score` (number), `budget_mentioned` (boolean), `primary_topic` (string), and `suggested_action` (string).

3.  **Data Processing:**
    *   Add a **Code Node** to merge the AI outputs with the original email metadata (sender email, subject). Use JavaScript to clean the email address (extracting the string between `< >`).
    *   Add a **Google Sheets** node set to `Append`. Map the properties from the Code Node to your sheet columns.

4.  **Salesforce Integration:**
    *   **Custom Fields:** In Salesforce, create custom fields on the Lead object (e.g., `Urgency_Score__c`).
    *   **Upsert Node:** Add a Salesforce node. Set Action to `Upsert`, Resource to `Lead`, and External ID to `Email`.
    *   **Conditional Logic:** Add an **IF Node** to check if `sentiment == Positive` AND `urgency >= 7`.
    *   **Opportunity/Task Nodes:** Add Salesforce nodes to create an Opportunity (on the "True" branch) and a Task (linked to the Lead ID).

5.  **Routing Logic:**
    *   Add a **Switch Node** branching on the `sentiment` value.
    *   Add **Gate Nodes** (Evaluation check) to each branch to prevent spamming during tests.
    *   Add **Slack** and **Gmail** (Send) nodes to notify the team based on the branch.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Contact for consulting and support | [Tally Form](https://tally.so/r/EkKGgB) |
| Developer Profile | [LinkedIn - Milo Bravo](https://www.linkedin.com/in/milobravo/) |
| Recommended Salesforce Setup | Create 9 custom Lead fields: `Sentiment__c`, `Sentiment_Confidence__c`, `Primary_Topic__c`, `Lead_Intent__c`, `Urgency_Score__c`, `Budget_Mentioned__c`, `Event_Referenced__c`, `Email_Domain__c`, `Last_Email_Subject__c` |
| Performance Tip | Use `Gemini 2.0 Flash` for the best balance of speed and cost in lead extraction. |