Qualify and score leads in Airtable with OpenAI and Slack alerts

https://n8nworkflows.xyz/workflows/qualify-and-score-leads-in-airtable-with-openai-and-slack-alerts-13746


# Qualify and score leads in Airtable with OpenAI and Slack alerts

This document provides a technical breakdown of the n8n workflow: **Qualify and score leads in Airtable with OpenAI and Slack alerts**.

### 1. Workflow Overview
The purpose of this workflow is to automate the lead qualification process by combining AI-driven analysis with a rule-based scoring engine. It monitors an Airtable base for new leads, enriches them using OpenAI (GPT), calculates a hybrid score, updates the CRM, and alerts the sales team via Slack when high-priority ("Hot") leads are detected.

**Logical Blocks:**
*   **1.1 Data Ingestion & Filtering:** Triggers on new records and ensures only unprocessed leads enter the pipeline.
*   **1.2 AI Qualification:** Uses OpenAI to evaluate qualitative data (e.g., job title, company description) and parse the results.
*   **1.3 Hybrid Scoring Engine:** A custom logic block that merges AI insights with quantitative data to produce a final lead score.
*   **1.4 CRM Synchronization:** Updates the original Airtable record with the calculated score and qualification status.
*   **1.5 Sales Notification:** Filters for high-intent leads and sends real-time alerts to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Filtering
*   **Overview:** Captures new lead data from Airtable and performs initial cleaning and deduplication checks.
*   **Nodes Involved:** `Trigger: New Lead Created`, `Normalize & Standardize Lead Data`, `Guard: Skip Already Processed Leads`, `Exit: Already Processed`.
*   **Node Details:**
    *   **Trigger: New Lead Created (Airtable Trigger):** Watches a specific Airtable base/table for new rows. *Requires Airtable Personal Access Token.*
    *   **Normalize & Standardize Lead Data (Set):** Maps raw Airtable columns to standardized internal variables (e.g., `email`, `company_size`, `intent_description`).
    *   **Guard: Skip Already Processed Leads (If):** Checks if a "Processed" flag or "Score" already exists to prevent infinite loops or double-scoring.
    *   **Exit: Already Processed (No-Op):** Termination point for records that do not need processing.

#### 2.2 AI Qualification
*   **Overview:** Sends lead context to OpenAI to categorize the lead and extract sentiment/intent.
*   **Nodes Involved:** `Prepare AI Scoring Context`, `AI Lead Qualification (GPT)`, `Validate & Parse AI Response`.
*   **Node Details:**
    *   **Prepare AI Scoring Context (Set):** Constructs a prompt-ready string containing the lead's qualitative information.
    *   **AI Lead Qualification (GPT) (OpenAI):** Uses a Chat Model (e.g., GPT-4o) to analyze the lead. Configuration usually involves a System Prompt asking for a JSON-formatted response containing `fit_score` and `reasoning`.
    *   **Validate & Parse AI Response (Code):** Uses JavaScript to ensure the AI's response is valid JSON. If the AI returns malformed text, this node handles the error to prevent workflow crash.

#### 2.3 Error Handling & Fallback
*   **Overview:** Ensures the workflow continues even if the AI service fails or returns an invalid response.
*   **Nodes Involved:** `AI Response Validation`, `Merge Valid & Fallback Paths`, `Alert: AI Scoring Failure`.
*   **Node Details:**
    *   **AI Response Validation (If):** Routes the flow based on whether the parsing in the previous step succeeded.
    *   **Alert: AI Scoring Failure (Slack):** Sends a notification to an internal channel if the AI step fails, allowing for manual intervention.
    *   **Merge Valid & Fallback Paths (Merge):** Recombines the data stream so the scoring engine receives either the AI data or default "neutral" values.

#### 2.4 Hybrid Scoring Engine
*   **Overview:** The central logic where qualitative AI scores and quantitative firmographic data are combined.
*   **Nodes Involved:** `Hybrid Lead Scoring Engine`.
*   **Node Details:**
    *   **Technical Role:** Code Node (JavaScript).
    *   **Configuration:** Implements a weighted formula (e.g., `Final Score = (AI_Fit * 0.6) + (Company_Size_Weight * 0.4)`).
    *   **Edge Cases:** Handles missing data by applying default weights to ensure every lead gets a score.

#### 2.5 CRM Update & Sales Alert
*   **Overview:** Persists the results back to the source of truth and notifies the team of urgent opportunities.
*   **Nodes Involved:** `Prepare CRM Update Payload`, `Update Lead Record (CRM)`, `Prepare Hot Lead Notification`, `Decision: Is High-Intent Lead?`, `Notify Sales Team – Hot Lead`, `Exit: No Sales Notification`.
*   **Node Details:**
    *   **Update Lead Record (CRM) (Airtable):** Updates the original record ID with the new `Lead Score`, `Qualification Tier`, and `AI Summary`.
    *   **Decision: Is High-Intent Lead? (If):** Checks if the `Final Score` exceeds a specific threshold (e.g., > 80).
    *   **Notify Sales Team – Hot Lead (Slack):** Sends a formatted message with a link to the Airtable record and a summary of why the lead is considered "Hot".

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Trigger: New Lead Created | Airtable Trigger | Event Trigger | None | Normalize & Standardize Lead Data | |
| Normalize & Standardize Lead Data | Set | Data Formatting | Trigger: New Lead Created | Guard: Skip Already Processed Leads | |
| Guard: Skip Already Processed Leads | If | Data Filtering | Normalize & Standardize Lead Data | Prepare AI Scoring Context, Exit: Already Processed | |
| Exit: Already Processed | No-Op | Termination | Guard: Skip Already Processed Leads | None | |
| Prepare AI Scoring Context | Set | AI Prep | Guard: Skip Already Processed Leads | AI Lead Qualification (GPT) | |
| AI Lead Qualification (GPT) | OpenAI | AI Analysis | Prepare AI Scoring Context | Validate & Parse AI Response | |
| Validate & Parse AI Response | Code | Data Parsing | AI Lead Qualification (GPT) | AI Response Validation | |
| AI Response Validation | If | Error Handling | Validate & Parse AI Response | Merge Valid & Fallback Paths, Alert: AI Scoring Failure | |
| Alert: AI Scoring Failure | Slack | Notification | AI Response Validation | None | |
| Merge Valid & Fallback Paths | Merge | Data Flow | AI Response Validation | Hybrid Lead Scoring Engine | |
| Hybrid Lead Scoring Engine | Code | Logic/Math | Merge Valid & Fallback Paths | Prepare CRM Update Payload | |
| Prepare CRM Update Payload | Set | Data Mapping | Hybrid Lead Scoring Engine | Update Lead Record (CRM) | |
| Update Lead Record (CRM) | Airtable | Database Update | Prepare CRM Update Payload | Prepare Hot Lead Notification | |
| Prepare Hot Lead Notification | Set | Alert Prep | Update Lead Record (CRM) | Decision: Is High-Intent Lead? | |
| Decision: Is High-Intent Lead? | If | Logic Gate | Prepare Hot Lead Notification | Notify Sales Team – Hot Lead, Exit: No Sales Notification | |
| Notify Sales Team – Hot Lead | Slack | Notification | Decision: Is High-Intent Lead? | None | |
| Exit: No Sales Notification | No-Op | Termination | Decision: Is High-Intent Lead? | None | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Airtable Setup:** Create a base with fields for `Email`, `Company`, `Description`, `Lead Score` (Number), and `Status` (Select).
2.  **Trigger:** Add an **Airtable Trigger** node. Select "On Record Created". Connect your Airtable account via OAuth2 or PAT.
3.  **Data Cleaning:** Add a **Set** node to map your Airtable fields to generic keys like `lead_description`.
4.  **Filtering:** Add an **If** node to check if `Lead Score` is empty. If false, route to a **No-Op** node.
5.  **AI Configuration:**
    *   Add a **Set** node to define the prompt for the AI.
    *   Add an **OpenAI** node (Chat Model). Provide an API Key. Use a prompt like: *"Analyze this lead: {{description}}. Return JSON: {score: 0-10, reason: string}"*.
6.  **Parsing:** Add a **Code** node with `JSON.parse()` to turn the OpenAI string response into selectable fields.
7.  **Logic Engine:** Add a **Code** node. Write a script to combine the AI score with other fields (e.g., if `company_size` > 500, add 20 points).
8.  **Database Update:** Add an **Airtable** node. Set the action to "Update". Map the `ID` from the trigger and the `Score` from the Code node.
9.  **Notification Logic:** Add an **If** node to check if the final score is > 80.
10. **Slack Setup:** Add a **Slack** node. Choose "Send Message". Use expressions to include the Lead Name and Score in the text. *Requires Slack OAuth2 or Webhook.*

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure Airtable API limits are respected if processing high volumes. | [Airtable API Documentation](https://support.airtable.com/hc/en-us/articles/230893488) |
| OpenAI GPT-4o is recommended for more accurate qualitative analysis. | [OpenAI Model Overview](https://platform.openai.com/docs/models) |
| Use n8n "Error Trigger" workflows to monitor this lead scoring process at scale. | General Best Practice |