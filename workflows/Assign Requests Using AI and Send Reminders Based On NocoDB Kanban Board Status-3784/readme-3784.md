Assign Requests Using AI and Send Reminders Based On NocoDB Kanban Board Status

https://n8nworkflows.xyz/workflows/assign-requests-using-ai-and-send-reminders-based-on-nocodb-kanban-board-status-3784


# Assign Requests Using AI and Send Reminders Based On NocoDB Kanban Board Status

### 1. Workflow Overview

This workflow automates incident management for a support project manager using a NocoDB Kanban board enhanced with AI-driven prioritization and automated notifications. It targets teams managing SLA-like agreements, aiming to streamline communication between clients and developers by automatically categorizing incidents, assigning priorities, and sending timely reminders.

The workflow is logically divided into two main blocks:

- **1.1 Incident Intake and AI-Based Categorization**  
  Triggered by an incident form submission, this block retrieves incident definitions from NocoDB, sends the user’s incident description to an AI agent for category assignment, and inserts the enriched incident into the Kanban board.

- **1.2 Scheduled Task Status Monitoring and Notifications**  
  Triggered by a schedule or manual trigger, this block checks for unpicked or overdue tasks on the Kanban board and sends reminder emails and Slack messages to clients and assignees accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Incident Intake and AI-Based Categorization

**Overview:**  
This block handles new incident submissions via a form, enriches the incident data by comparing it with predefined incident definitions using AI, and inserts the categorized incident into the NocoDB Kanban board.

**Nodes Involved:**  
- Incident Form (disabled)  
- Get incident definitions  
- Aggregate for AI parsing  
- OpenAI Chat Model1  
- Assign Category (LangChain Agent)  
- Structure Output Todoist Ready1  
- Format for Noco  
- Insert Incident  
- Sticky Note (Incident Form)  
- Sticky Note1 (Parse Incidents)

**Node Details:**

- **Incident Form**  
  - *Type:* Form Trigger  
  - *Role:* Entry point for incident data submission (disabled by default)  
  - *Configuration:* Collects Email, Incident Description, and Incident Desired Category (dropdown with predefined options)  
  - *Input/Output:* Outputs form data to "Get incident definitions" node  
  - *Edge Cases:* Disabled by default; if replaced by email/webhook, references must be updated accordingly.

- **Get incident definitions**  
  - *Type:* NocoDB Node (Get All)  
  - *Role:* Fetches incident definitions from NocoDB table including Title, Definition, Response time, Resolution time, Default assignee  
  - *Configuration:* Uses NocoDB API token credential; returns all records  
  - *Input:* Triggered by Incident Form  
  - *Output:* Passes data to "Aggregate for AI parsing"  
  - *Edge Cases:* API token invalid or table misconfigured may cause failure.

- **Aggregate for AI parsing**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all incident definition items into a single JSON array for AI consumption  
  - *Input:* Incident definitions data  
  - *Output:* Passes aggregated data to "Assign Category"  
  - *Edge Cases:* Empty definitions table leads to empty input for AI.

- **OpenAI Chat Model1**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides AI language model backend (GPT-4o-mini) for LangChain agent  
  - *Configuration:* Uses OpenAI API credential  
  - *Input:* Receives prompt from "Assign Category" node  
  - *Output:* Returns AI response to "Assign Category"  
  - *Edge Cases:* API rate limits, invalid credentials, or model unavailability.

- **Assign Category**  
  - *Type:* LangChain Agent  
  - *Role:* Sends incident description and definitions to AI to assign category, response time, resolution time, and default assignee  
  - *Configuration:* Custom prompt with embedded incident definitions and user input; expects structured JSON output  
  - *Input:* Aggregated definitions and incident description from "On schedule or during flow" node (note: expression references "Incident Description" from the trigger)  
  - *Output:* Passes structured AI output to "Format for Noco"  
  - *Edge Cases:* AI output parsing errors, unexpected output format.

- **Structure Output Todoist Ready1**  
  - *Type:* LangChain Output Parser Structured  
  - *Role:* Parses AI output to enforce JSON schema with fields: category, response_time, resolution_time, default_assignee  
  - *Input:* AI raw output from "Assign Category"  
  - *Output:* Structured JSON for further processing  
  - *Edge Cases:* Parsing failures if AI output deviates from schema.

- **Format for Noco**  
  - *Type:* Set Node  
  - *Role:* Maps AI output and form data into fields matching NocoDB Tasks table schema  
  - *Configuration:* Assigns fields such as email, message, expected category, assigned category, status (default "todo"), expected response and resolution times (calculated as current time plus AI-provided hours)  
  - *Input:* AI parsed output and original form data  
  - *Output:* Data ready for insertion into NocoDB  
  - *Edge Cases:* Incorrect date/time calculations or missing AI fields.

- **Insert Incident**  
  - *Type:* NocoDB Node (Create)  
  - *Role:* Inserts the formatted incident record into the Kanban board table in NocoDB  
  - *Configuration:* Uses NocoDB API token credential  
  - *Input:* Data from "Format for Noco"  
  - *Output:* Triggers "On schedule or during flow" node for further processing  
  - *Edge Cases:* API failures, table misconfiguration.

- **Sticky Note (Incident Form)**  
  - *Type:* Sticky Note  
  - *Role:* Documentation note explaining the Incident Form trigger and customization hints.

- **Sticky Note1 (Parse Incidents)**  
  - *Type:* Sticky Note  
  - *Role:* Documentation note describing AI parsing and assignment logic.

---

#### 2.2 Scheduled Task Status Monitoring and Notifications

**Overview:**  
This block periodically checks the Kanban board for tasks that are either not picked up after the expected response time or not completed by the expected resolution time. It sends reminder emails and Slack messages to clients and assignees to keep everyone informed.

**Nodes Involved:**  
- Check status every day (disabled)  
- When clicking ‘Test workflow’ (manual trigger)  
- On schedule or during flow (NoOp)  
- Get Unpicked Tasks  
- Task is not picked up after expected response (If)  
- Send email to client  
- If there is asignee email (If)  
- Send email to asignee  
- Get Unfinished Tasks  
- Task is not complete in expected time (If)  
- Send email to client1  
- If there is assignee slack (If)  
- Slack to assignee  
- Send another email to asignee  
- Sticky Note2 (Stay Informed)  
- Sticky Note3 (Task incomplete)  
- Sticky Note4 (Trigger Task Check Daily)

**Node Details:**

- **Check status every day**  
  - *Type:* Schedule Trigger (disabled)  
  - *Role:* Intended to trigger daily at 9 AM to start task status checks  
  - *Edge Cases:* Disabled by default; enabling requires credential and schedule setup.

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual triggering of the status check flow for testing.

- **On schedule or during flow**  
  - *Type:* NoOp  
  - *Role:* Central node that branches to "Get Unpicked Tasks" and "Get Unfinished Tasks"  
  - *Input:* Triggered by schedule or manual trigger  
  - *Output:* Feeds two parallel branches for task checking.

- **Get Unpicked Tasks**  
  - *Type:* NocoDB Get All  
  - *Role:* Fetches up to 5 tasks with status "todo" (not picked up)  
  - *Configuration:* Filters by status = "todo"  
  - *Output:* Passes tasks to "Task is not picked up after expected response"  
  - *Edge Cases:* API errors or empty results.

- **Task is not picked up after expected response**  
  - *Type:* If Node  
  - *Role:* Checks if the current time is past the task's Expected Response date and status is "todo"  
  - *Output:* True branch triggers notifications, false branch ends flow for that task  
  - *Edge Cases:* Date parsing errors, missing fields.

- **Send email to client**  
  - *Type:* Email Send  
  - *Role:* Sends apology and update email to client about unpicked task  
  - *Configuration:* Uses SMTP credential; email text and subject predefined  
  - *Input:* True branch from previous If node  
  - *Edge Cases:* SMTP failures, invalid email addresses.

- **If there is asignee email**  
  - *Type:* If Node  
  - *Role:* Checks if assignee email is present before sending notification  
  - *Output:* True branch triggers "Send email to asignee"  
  - *Edge Cases:* Missing assignee email.

- **Send email to asignee**  
  - *Type:* Email Send  
  - *Role:* Notifies assignee about outstanding task to pick up  
  - *Configuration:* Uses SMTP credential  
  - *Edge Cases:* SMTP failures, invalid assignee email.

- **Get Unfinished Tasks**  
  - *Type:* NocoDB Get All  
  - *Role:* Fetches up to 5 tasks with status "todo" (unfinished tasks)  
  - *Output:* Passes tasks to "Task is not complete in expected time"  
  - *Edge Cases:* API errors or empty results.

- **Task is not complete in expected time**  
  - *Type:* If Node  
  - *Role:* Checks if current time is past Expected Resolution and status is not "done"  
  - *Output:* True branch triggers notifications, false branch ends flow  
  - *Edge Cases:* Date parsing errors, missing fields.

- **Send email to client1**  
  - *Type:* Email Send  
  - *Role:* Sends apology and update email to client about overdue task  
  - *Configuration:* Uses SMTP credential  
  - *Edge Cases:* SMTP failures, invalid email addresses.

- **If there is assignee slack**  
  - *Type:* If Node  
  - *Role:* Checks if assignee Slack username is present  
  - *Output:* True branch triggers Slack message; false branch triggers fallback email to assignee  
  - *Edge Cases:* Missing Slack username.

- **Slack to assignee**  
  - *Type:* Slack Node  
  - *Role:* Sends Slack message to assignee about overdue task  
  - *Configuration:* Uses Slack OAuth2 credential; message text predefined  
  - *Edge Cases:* Slack API errors, invalid username.

- **Send another email to asignee**  
  - *Type:* Email Send  
  - *Role:* Fallback notification email to assignee if Slack username missing  
  - *Configuration:* Uses SMTP credential  
  - *Edge Cases:* SMTP failures.

- **Sticky Note2 (Stay Informed)**  
  - *Type:* Sticky Note  
  - *Role:* Documentation about informing client and developer, with suggestion to replace emails with Slack messages if preferred.

- **Sticky Note3 (Task incomplete)**  
  - *Type:* Sticky Note  
  - *Role:* Documentation about notifying client and developer for overdue tasks.

- **Sticky Note4 (Trigger Task Check Daily)**  
  - *Type:* Sticky Note  
  - *Role:* Documentation recommending daily schedule frequency to avoid excessive notifications.

---

### 3. Summary Table

| Node Name                          | Node Type                        | Functional Role                                   | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                                                                      |
|-----------------------------------|---------------------------------|-------------------------------------------------|----------------------------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Incident Form                     | Form Trigger                    | Entry point for incident submission              | -                                | Get incident definitions                     | ## Incident Form: Triggered by form; can be replaced with email/webhook (update references accordingly)          |
| Get incident definitions          | NocoDB Get All                 | Fetch incident definitions from NocoDB           | Incident Form                   | Aggregate for AI parsing                      | ## Parse Incidents: AI compares definitions with user input                                                     |
| Aggregate for AI parsing          | Aggregate                      | Aggregate definitions into JSON array             | Get incident definitions         | Assign Category                              | ## Parse Incidents: AI compares definitions with user input                                                     |
| OpenAI Chat Model1                | LangChain OpenAI Chat Model    | Provides GPT-4o-mini model for AI processing      | Assign Category (ai_languageModel) | Assign Category                              |                                                                                                                 |
| Assign Category                  | LangChain Agent                | AI assigns category, response/resolution times, assignee | Aggregate for AI parsing, OpenAI Chat Model1 | Format for Noco                              | ## Parse Incidents: AI compares definitions with user input                                                     |
| Structure Output Todoist Ready1  | LangChain Output Parser        | Parses AI output to structured JSON                | Assign Category                 | Assign Category (ai_outputParser)            |                                                                                                                 |
| Format for Noco                  | Set                           | Formats data for NocoDB insertion                   | Assign Category                 | Insert Incident                              | ## Parse Incidents: AI compares definitions with user input                                                     |
| Insert Incident                 | NocoDB Create                 | Inserts new incident into Kanban board              | Format for Noco                 | On schedule or during flow                    |                                                                                                                 |
| On schedule or during flow       | NoOp                          | Central node for scheduled/manual task checks      | Insert Incident, Check status every day, When clicking ‘Test workflow’ | Get Unpicked Tasks, Get Unfinished Tasks |                                                                                                                 |
| Get Unpicked Tasks               | NocoDB Get All                | Retrieves tasks not picked up (status "todo")      | On schedule or during flow       | Task is not picked up after expected response |                                                                                                                 |
| Task is not picked up after expected response | If                            | Checks if task response time exceeded and status "todo" | Get Unpicked Tasks              | Send email to client, If there is asignee email |                                                                                                                 |
| Send email to client             | Email Send                    | Notifies client about unpicked task                 | Task is not picked up after expected response | -                                           | ## Stay Informed: Inform client and developer; can replace with Slack messages                                  |
| If there is asignee email        | If                            | Checks for presence of assignee email               | Task is not picked up after expected response | Send email to asignee                        | ## Stay Informed: Inform client and developer; can replace with Slack messages                                  |
| Send email to asignee            | Email Send                    | Notifies assignee about unpicked task               | If there is asignee email        | -                                           | ## Stay Informed: Inform client and developer; can replace with Slack messages                                  |
| Get Unfinished Tasks             | NocoDB Get All                | Retrieves tasks not completed (status "todo")       | On schedule or during flow       | Task is not complete in expected time         |                                                                                                                 |
| Task is not complete in expected time | If                            | Checks if task resolution time exceeded and status not "done" | Get Unfinished Tasks            | Send email to client1, If there is assignee slack |                                                                                                                 |
| Send email to client1            | Email Send                    | Notifies client about overdue task                   | Task is not complete in expected time | -                                           | ## Task incomplete: Notify client and developer about overdue tasks                                            |
| If there is assignee slack       | If                            | Checks for presence of assignee Slack username       | Task is not complete in expected time | Slack to assignee, Send another email to asignee | ## Task incomplete: Notify client and developer about overdue tasks                                            |
| Slack to assignee               | Slack                         | Sends Slack message to assignee about overdue task  | If there is assignee slack       | -                                           | ## Task incomplete: Notify client and developer about overdue tasks                                            |
| Send another email to asignee    | Email Send                    | Fallback email notification to assignee if no Slack | If there is assignee slack       | -                                           | ## Task incomplete: Notify client and developer about overdue tasks                                            |
| Sticky Note                     | Sticky Note                   | Documentation on Incident Form trigger               | -                                | -                                           | ## Incident Form: Triggered by form; can be replaced with email/webhook (update references accordingly)          |
| Sticky Note1                    | Sticky Note                   | Documentation on AI parsing and assignment           | -                                | -                                           | ## Parse Incidents: AI compares definitions with user input                                                     |
| Sticky Note2                    | Sticky Note                   | Documentation on informing client and developer      | -                                | -                                           | ## Stay Informed: Inform client and developer; can replace with Slack messages                                  |
| Sticky Note3                    | Sticky Note                   | Documentation on notifying about overdue tasks       | -                                | -                                           | ## Task incomplete: Notify client and developer about overdue tasks                                            |
| Sticky Note4                    | Sticky Note                   | Documentation on scheduling frequency recommendation | -                                | -                                           | ## Trigger Task Check Daily: Recommend daily schedule to avoid excessive notifications                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Incident Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form with fields:  
     - Email (type: email, required)  
     - Incident Description (type: textarea, required)  
     - Incident Desired Category (type: dropdown, options: Critical, Major, Medium, Minor, Support, Feature; required)  
   - Set form title and description accordingly.  
   - Note: This node is disabled by default; enable if using form input.

2. **Add NocoDB Node to Get Incident Definitions**  
   - Operation: Get All  
   - Table: Incident definitions table (e.g., "mt94l49b6zocsxy")  
   - Fields: Title, Definition, Response time, Resolution time, Default assignee  
   - Use NocoDB API token credential.

3. **Add Aggregate Node**  
   - Operation: Aggregate All Item Data  
   - Input: Incident definitions from previous node  
   - Purpose: Aggregate all definitions into one JSON array for AI.

4. **Add LangChain OpenAI Chat Model Node**  
   - Model: gpt-4o-mini  
   - Use OpenAI API credential.

5. **Add LangChain Agent Node (Assign Category)**  
   - Prompt: Provide incident definitions and user incident description to AI, requesting category, response time, resolution time, and default assignee.  
   - Input variables:  
     - Definitions aggregated JSON  
     - Incident Description from form trigger  
   - Output: Structured JSON with category, response_time, resolution_time, default_assignee.

6. **Add LangChain Output Parser Structured Node**  
   - JSON schema example:  
     ```json
     {
       "category": "critical",
       "response_time": "1",
       "resolution_time": "8",
       "default_assignee": "email@example.com"
     }
     ```  
   - Input: AI raw output from LangChain Agent.

7. **Add Set Node (Format for Noco)**  
   - Map fields:  
     - email: from form input  
     - message: incident description from form input  
     - expected category: incident desired category from form input  
     - assigned category: AI output category  
     - status: "todo" (default)  
     - Expected Response: current time plus AI response_time (hours)  
     - Expected Resolution: current time plus AI resolution_time (hours)  
   - Use expressions to calculate dates.

8. **Add NocoDB Node (Insert Incident)**  
   - Operation: Create  
   - Table: Kanban board tasks table (e.g., "mwh33g1yyeg9z6k")  
   - Data: Auto map from previous Set node  
   - Use NocoDB API token credential.

9. **Add NoOp Node (On schedule or during flow)**  
   - Acts as a central branching point for scheduled and manual triggers.

10. **Add Schedule Trigger Node (Check status every day)**  
    - Configure to trigger daily at 9 AM (disabled by default).  
    - Connect to NoOp node.

11. **Add Manual Trigger Node (When clicking ‘Test workflow’)**  
    - Connect to NoOp node for manual testing.

12. **Add NocoDB Node (Get Unpicked Tasks)**  
    - Operation: Get All  
    - Table: Kanban board tasks table  
    - Filter: status equals "todo"  
    - Limit: 5  
    - Use NocoDB API token credential.

13. **Add If Node (Task is not picked up after expected response)**  
    - Condition 1: Expected Response < current time  
    - Condition 2: status equals "todo"  
    - Both must be true.

14. **Add Email Send Node (Send email to client)**  
    - To: client email from task data  
    - From: support@example.com  
    - Subject: "Your task is important to us"  
    - Text: apology message about unpicked task  
    - Use SMTP credential.

15. **Add If Node (If there is asignee email)**  
    - Condition: assignee email field is not empty.

16. **Add Email Send Node (Send email to asignee)**  
    - To: assignee email  
    - From: support@example.com  
    - Subject: "Your task is important to us"  
    - Text: notification about outstanding task  
    - Use SMTP credential.

17. **Add NocoDB Node (Get Unfinished Tasks)**  
    - Operation: Get All  
    - Table: Kanban board tasks table  
    - Filter: status equals "todo"  
    - Limit: 5  
    - Use NocoDB API token credential.

18. **Add If Node (Task is not complete in expected time)**  
    - Condition 1: Expected Resolution < current time  
    - Condition 2: status not equals "done"  
    - Both must be true.

19. **Add Email Send Node (Send email to client1)**  
    - To: client email  
    - From: support@example.com  
    - Subject: "Your task is important to us"  
    - Text: apology message about overdue task  
    - Use SMTP credential.

20. **Add If Node (If there is assignee slack)**  
    - Condition: assignee slack username is not empty.

21. **Add Slack Node (Slack to assignee)**  
    - User: assignee slack username  
    - Text: notification about unfinished task  
    - Use Slack OAuth2 credential.

22. **Add Email Send Node (Send another email to asignee)**  
    - To: assignee email (fallback if no Slack)  
    - From: support@example.com  
    - Subject: "Your task is important to us"  
    - Text: notification about unfinished task  
    - Use SMTP credential.

23. **Connect nodes accordingly:**  
    - NoOp node branches to Get Unpicked Tasks and Get Unfinished Tasks nodes in parallel.  
    - Each task retrieval node connects to its respective If node for condition checks.  
    - True branches from If nodes connect to notification nodes as described.

24. **Add Sticky Notes**  
    - Add documentation sticky notes at appropriate places to explain form trigger, AI parsing, notification logic, and scheduling recommendations.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Incident naming must be consistent across NocoDB tables and n8n nodes: Incident definitions "Title", Tasks table single select fields "expected category" and "assigned category", and Incident Form "Incident Desired Category". | Workflow setup instructions.                                                                           |
| Use Kanban board view in NocoDB stacked by "status" field for visual task management.                            | Workflow setup instructions.                                                                           |
| Consider replacing email notifications with Slack messages if preferred; workflow nodes support both.           | Sticky Note2 content.                                                                                   |
| Running task status checks more than once daily may cause excessive client notifications.                        | Sticky Note4 content.                                                                                   |
| For alternative triggers (e.g., email trigger), update node references and field mappings accordingly.           | Workflow description notes.                                                                             |
| OpenAI API key and Slack OAuth2 credentials must be configured and valid for AI and Slack nodes respectively.    | Credential setup requirement.                                                                           |
| SMTP credentials required for sending emails; ensure proper configuration and permissions.                        | Credential setup requirement.                                                                           |
| For further customization or support, visit the author’s profile or contact for dedicated software development.  | Workflow description closing remarks.                                                                  |

---

This document provides a comprehensive reference for understanding, modifying, and reproducing the "Noco Kanban Board with AI Prioritization" workflow. It covers all nodes, their roles, configurations, and interconnections, ensuring clarity for advanced users and automation agents alike.