Automate Gmail Lead Follow-Up with OpenAI GPT-4O, HubSpot, Slack & Google Sheets

https://n8nworkflows.xyz/workflows/automate-gmail-lead-follow-up-with-openai-gpt-4o--hubspot--slack---google-sheets-7680


# Automate Gmail Lead Follow-Up with OpenAI GPT-4O, HubSpot, Slack & Google Sheets

### 1. Workflow Overview

This workflow automates lead follow-up management by integrating Gmail, OpenAI GPT-4o, HubSpot CRM, Slack, and Google Sheets. It targets sales and marketing teams who receive inbound lead emails and want to automatically analyze, prioritize, and track follow-up actions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception (Gmail Trigger & Validation):** Captures new lead emails labeled in Gmail and validates the presence of data.
- **1.2 Data Normalization:** Standardizes key email fields for downstream processing.
- **1.3 AI Processing:** Uses OpenAI GPT-4o to analyze the lead’s message for sentiment, intent, urgency, priority, and next action, then parses and enriches the data.
- **1.4 Decision Logic:** Evaluates AI results to determine if follow-up actions are needed.
- **1.5 Follow-Up Actions:** Creates HubSpot follow-up tasks, notifies the sales team via Slack, and logs details into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for new Gmail messages tagged with a specific label, validates the incoming data to ensure the email content exists, and prepares for processing.

**Nodes Involved:**  
- 📧 Gmail: New Lead Response  
- 🔀 Check if Data Exists  

**Node Details:**  

- **📧 Gmail: New Lead Response**  
  - Type: Gmail Trigger  
  - Role: Triggers workflow on new emails with a designated Gmail label every minute.  
  - Configuration: Watches for emails with a specific Gmail label ID; polling interval set to every minute.  
  - Inputs: Gmail inbox filtered by label.  
  - Outputs: Email data JSON, including sender, subject, snippet, and timestamp.  
  - Edge Cases: OAuth token expiration, Gmail API rate limits, no new emails.  
  - Credentials: Gmail OAuth2 required.

- **🔀 Check if Data Exists**  
  - Type: If node  
  - Role: Validates existence of incoming Gmail data before proceeding.  
  - Configuration: Checks if the JSON data from Gmail node exists (non-empty).  
  - Inputs: Output from Gmail trigger.  
  - Outputs: Proceeds only if data exists; otherwise stops workflow.  
  - Edge Cases: Empty or malformed Gmail data could halt workflow.  
  - Version: v2 or later recommended for strict validation.

---

#### 2.2 Data Normalization

**Overview:**  
Standardizes email fields into consistent variable names to simplify downstream processing and ensure uniform data structure.

**Nodes Involved:**  
- 🔧 Normalize Gmail Data  

**Node Details:**  

- **🔧 Normalize Gmail Data**  
  - Type: Set node  
  - Role: Maps Gmail JSON fields into normalized fields: leadEmail, subject, message, source, receivedAt.  
  - Configuration: Assigns From → leadEmail, Subject → subject, snippet → message, adds source as "Gmail", and internalDate → receivedAt.  
  - Inputs: Output of data existence check node.  
  - Outputs: Normalized JSON object with defined keys.  
  - Edge Cases: Missing expected Gmail fields; timestamp format consistency.  
  - Version: v3.3 used, supports advanced assignments.

---

#### 2.3 AI Processing

**Overview:**  
Analyzes the normalized lead email using OpenAI GPT-4o to extract insights such as sentiment, intent, urgency, next recommended action, summary, and priority. Parses the AI JSON response and merges it with original lead data, adding flags for follow-up decisions.

**Nodes Involved:**  
- 🧠 Analyze Lead Response (AI)  
- OpenAI Chat Model  
- 📋 Parse AI Analysis Results  

**Node Details:**  

- **OpenAI Chat Model**  
  - Type: n8n Langchain OpenAI Chat node  
  - Role: Provides the GPT-4o AI model interface for text analysis.  
  - Configuration: Uses model "gpt-4o-mini" with default options; no additional prompt tweaks.  
  - Inputs: Text prompt from Analyze Lead Response node.  
  - Outputs: Raw AI chat response.  
  - Credentials: OpenAI API key required.  
  - Edge Cases: API rate limits, malformed responses, network timeouts.

- **🧠 Analyze Lead Response (AI)**  
  - Type: Langchain Agent (AI prompt node)  
  - Role: Constructs prompt with normalized lead data requesting detailed JSON analysis (sentiment, intent, urgency, etc.)  
  - Configuration: Prompt explicitly asks for 6 fields in JSON format; injects leadEmail, subject, message dynamically.  
  - Inputs: Normalized lead data from previous block.  
  - Outputs: Text prompt to OpenAI Chat Model node.  
  - Edge Cases: Prompt formatting errors, unexpected AI output structure.

- **📋 Parse AI Analysis Results**  
  - Type: Code node (JavaScript)  
  - Role: Parses AI JSON response, cleans formatting, handles fallback on parse errors, and merges with lead data. Adds boolean flags `needsFollowUp` and `isHighPriority`.  
  - Configuration: Strips markdown backticks, attempts JSON parse, fallback to default analysis if parsing fails.  
  - Inputs: AI raw output and normalized lead data (referenced node).  
  - Outputs: Enriched JSON object for decision logic.  
  - Edge Cases: Malformed JSON, missing expected keys, unexpected AI text formatting.

---

#### 2.4 Decision Logic

**Overview:**  
Checks if the AI analysis indicates that follow-up is required. If yes, triggers follow-up actions; if no, the workflow ends.

**Nodes Involved:**  
- 🔍 Needs Follow-Up?  

**Node Details:**  

- **🔍 Needs Follow-Up?**  
  - Type: If node  
  - Role: Evaluates `needsFollowUp` boolean flag from AI parsed results.  
  - Configuration: Passes workflow forward if `needsFollowUp` is true; otherwise stops.  
  - Inputs: Parsed AI analysis JSON.  
  - Outputs: Triggers follow-up nodes on true branch.  
  - Edge Cases: Missing flag, edge cases where nextAction = "No Action" should halt follow-up.

---

#### 2.5 Follow-Up Actions

**Overview:**  
Performs the actual follow-up steps: creates a task in HubSpot CRM, sends a notification to the sales team via Slack, and logs the lead analysis to Google Sheets for tracking.

**Nodes Involved:**  
- 📝 HubSpot: Create Follow-up Task  
- 💬 Slack: Notify Sales Team  
- 📊 Log to Google Sheets  

**Node Details:**  

- **📝 HubSpot: Create Follow-up Task**  
  - Type: HubSpot node (engagement resource)  
  - Role: Creates a task linked to a contact in HubSpot with email and message details.  
  - Configuration: Task body set to lead message, subject set to email subject, linked to CONTACT type object.  
  - Inputs: AI parsed data with lead info.  
  - Outputs: HubSpot API response on task creation.  
  - Credentials: HubSpot App Token required.  
  - Edge Cases: API auth failure, invalid contact linkage, rate limits.

- **💬 Slack: Notify Sales Team**  
  - Type: Slack node (message)  
  - Role: Posts a Slack message to a sales channel summarizing lead insights: summary, email, priority, urgency, analysis date.  
  - Configuration: Sends plain text (mrkdwn=false) to a configured Slack channel via webhook.  
  - Inputs: AI parsed data.  
  - Credentials: Slack API token.  
  - Edge Cases: Webhook failures, channel access issues.

- **📊 Log to Google Sheets**  
  - Type: Google Sheets node  
  - Role: Appends a row to a specified Google Sheet with all relevant lead and analysis data for monitoring.  
  - Configuration: Defines columns mapping fields like Date, Email, Intent, Message, etc., and appends data to sheet gid=0.  
  - Inputs: AI parsed and enriched lead data.  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Sheet access permissions, rate limits, document ID or sheet name errors.

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                            | Input Node(s)                 | Output Node(s)                                  | Sticky Note                                                                                                                           |
|-----------------------------|-----------------------------|--------------------------------------------|------------------------------|------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| 📧 Gmail: New Lead Response | Gmail Trigger               | Triggers on new lead email in Gmail label  | None                         | 🔀 Check if Data Exists                         | Captures new lead responses from Gmail with a specific label every minute. Verifies presence of data and standardizes fields.         |
| 🔀 Check if Data Exists      | If                         | Validates presence of Gmail data            | 📧 Gmail: New Lead Response   | 🔧 Normalize Gmail Data                         | Captures new lead responses from Gmail with a specific label every minute. Verifies presence of data and standardizes fields.         |
| 🔧 Normalize Gmail Data      | Set                        | Normalizes and standardizes Gmail email fields | 🔀 Check if Data Exists       | 🧠 Analyze Lead Response (AI)                   | Captures new lead responses from Gmail with a specific label every minute. Verifies presence of data and standardizes fields.         |
| 🧠 Analyze Lead Response (AI)| Langchain Agent (AI prompt) | Builds AI prompt and processes lead data    | 🔧 Normalize Gmail Data       | 📋 Parse AI Analysis Results                    | Utilizes OpenAI to analyze the lead's message for sentiment, intent, urgency, next action, and priority. Parses and enriches data.     |
| OpenAI Chat Model            | Langchain OpenAI Chat Model| Calls GPT-4o to analyze lead message        | 🧠 Analyze Lead Response (AI) | 📋 Parse AI Analysis Results                    | Utilizes OpenAI to analyze the lead's message for sentiment, intent, urgency, next action, and priority. Parses and enriches data.     |
| 📋 Parse AI Analysis Results | Code                       | Parses AI JSON response, merges with lead data | OpenAI Chat Model            | 🔍 Needs Follow-Up?                             | Utilizes OpenAI to analyze the lead's message for sentiment, intent, urgency, next action, and priority. Parses and enriches data.     |
| 🔍 Needs Follow-Up?          | If                         | Decides if follow-up actions are needed     | 📋 Parse AI Analysis Results  | 📝 HubSpot: Create Follow-up Task, 💬 Slack: Notify Sales Team, 📊 Log to Google Sheets | Evaluates if the AI-recommended next action requires follow-up. Triggers all follow-up actions if true.                               |
| 📝 HubSpot: Create Follow-up Task | HubSpot Engagement Task | Creates CRM task for sales follow-up        | 🔍 Needs Follow-Up?           | None                                           | Once a lead needs follow-up, creates HubSpot task, notifies sales, logs data.                                                         |
| 💬 Slack: Notify Sales Team  | Slack Message              | Sends sales team notification with lead insights | 🔍 Needs Follow-Up?           | None                                           | Once a lead needs follow-up, creates HubSpot task, notifies sales, logs data.                                                         |
| 📊 Log to Google Sheets      | Google Sheets Append       | Logs lead and AI analysis for record keeping | 🔍 Needs Follow-Up?           | None                                           | Once a lead needs follow-up, creates HubSpot task, notifies sales, logs data.                                                         |
| Sticky Note                 | Sticky Note                | Documentation and comments                   | None                         | None                                           | Covers nodes: Gmail trigger, data validation, normalization.                                                                         |
| Sticky Note1                | Sticky Note                | Documentation and comments                   | None                         | None                                           | Covers nodes: AI analysis, OpenAI model, AI response parsing.                                                                         |
| Sticky Note2                | Sticky Note                | Documentation and comments                   | None                         | None                                           | Covers node: Needs Follow-Up? decision node.                                                                                          |
| Sticky Note3                | Sticky Note                | Documentation and comments                   | None                         | None                                           | Covers nodes: HubSpot task creation, Slack notification, Google Sheets logging.                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: Gmail Trigger  
   - Parameters: Set to trigger on new email labeled with your Gmail label ID.  
   - Poll interval: every minute  
   - Credentials: Connect your Gmail OAuth2 credentials.

2. **Create If Node "Check if Data Exists":**  
   - Type: If  
   - Condition: Check if the trigger node’s JSON data exists (Object existence check).  
   - Connect Gmail Trigger → this node.

3. **Create Set Node "Normalize Gmail Data":**  
   - Type: Set  
   - Assign fields:  
     - leadEmail = From field from Gmail JSON  
     - subject = Subject  
     - message = snippet  
     - source = "Gmail" (static string)  
     - receivedAt = internalDate timestamp  
   - Connect "Check if Data Exists" (true branch) → this node.

4. **Create Langchain Agent Node "Analyze Lead Response (AI)":**  
   - Type: Langchain Agent (AI prompt)  
   - Prompt:  
     ```
     Analyze this lead response and provide:

     1. Sentiment: (Positive/Neutral/Negative)
     2. Intent: (Interested/Not Interested/Needs Info/Ready to Buy/Objection)
     3. Urgency: (High/Medium/Low)
     4. Next Action: (Call/Email/Demo/Quote/No Action)
     5. Summary: Brief 1-2 sentence summary
     6. Priority: (Hot/Warm/Cold)

     Lead Email: {{ $json.leadEmail }}
     Subject: {{ $json.subject }}
     Message: {{ $json.message }}

     Respond in JSON format with keys: sentiment, intent, urgency, nextAction, summary, priority
     ```
   - Connect "Normalize Gmail Data" → this node.

5. **Create OpenAI Chat Model Node:**  
   - Type: Langchain OpenAI Chat  
   - Model: gpt-4o-mini  
   - Connect "Analyze Lead Response (AI)" → this node.  
   - Credentials: Provide your OpenAI API Key.

6. **Create Code Node "Parse AI Analysis Results":**  
   - Type: Code (JavaScript)  
   - Script:  
     - Strip markdown backticks from AI output.  
     - Parse JSON from cleaned string.  
     - On parse failure, fallback to default neutral analysis.  
     - Merge parsed analysis with original normalized lead data from "Normalize Gmail Data".  
     - Add flags: `needsFollowUp` (true if nextAction != "No Action"), `isHighPriority` (true if priority = Hot or urgency = High).  
   - Connect OpenAI Chat Model → this node.

7. **Create If Node "Needs Follow-Up?":**  
   - Type: If  
   - Condition: Check if `needsFollowUp` is true.  
   - Connect "Parse AI Analysis Results" → this node.

8. **Create HubSpot Node "Create Follow-up Task":**  
   - Type: HubSpot Engagement Task  
   - Parameters:  
     - Type: task  
     - Subject: from AI parsed subject  
     - Body: from AI parsed message  
     - For Object Type: CONTACT  
   - Credentials: HubSpot App Token  
   - Connect "Needs Follow-Up?" (true branch) → this node.

9. **Create Slack Node "Notify Sales Team":**  
   - Type: Slack  
   - Parameters:  
     - Text: summary, lead email, priority, urgency, analysis date from AI parsed data  
     - Channel: your designated Slack channel ID  
     - Mrkdwn: false  
   - Credentials: Slack API token  
   - Connect "Needs Follow-Up?" (true branch) → this node.

10. **Create Google Sheets Node "Log to Google Sheets":**  
    - Type: Google Sheets Append  
    - Document ID: your Google Sheets ID  
    - Sheet Name: Sheet gid=0 (or your sheet name)  
    - Columns: map Date, Email, Intent, Message, Subject, Summary, Urgency, Priority, Sentiment, NextAction from AI parsed data  
    - Credentials: Google Sheets OAuth2  
    - Connect "Needs Follow-Up?" (true branch) → this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow automates lead follow-up integrating Gmail, OpenAI GPT-4o, HubSpot, Slack, Google Sheets.                   | Project scope                                                                                         |
| Gmail label ID, OAuth2 credentials, HubSpot App Token, Slack API token, OpenAI API key, Google Sheets ID must be set. | Credentials management                                                                                 |
| Sticky notes in workflow provide grouped node explanations and workflow logic insights.                              | Visual documentation inside n8n workflow                                                             |
| OpenAI prompt expects strictly formatted JSON to parse AI response successfully.                                     | Important for AI node stability                                                                       |
| HubSpot engagement tasks require existing contacts linked by email in HubSpot CRM.                                   | Integration constraint                                                                                 |
| Slack notifications use plain text (no markdown) for compatibility and clarity in sales channels.                    | Slack message formatting consideration                                                               |
| Google Sheets columns must match the defined schema exactly to avoid logging errors.                                 | Google Sheets integration best practice                                                              |
| Potential failure points: API rate limits, OAuth token expiration, malformed AI response, missing Gmail data.        | Monitor and handle node errors and workflow alerts                                                   |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.