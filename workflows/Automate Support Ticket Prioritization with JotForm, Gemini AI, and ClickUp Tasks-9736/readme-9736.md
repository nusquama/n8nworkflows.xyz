Automate Support Ticket Prioritization with JotForm, Gemini AI, and ClickUp Tasks

https://n8nworkflows.xyz/workflows/automate-support-ticket-prioritization-with-jotform--gemini-ai--and-clickup-tasks-9736


# Automate Support Ticket Prioritization with JotForm, Gemini AI, and ClickUp Tasks

### 1. Workflow Overview

This workflow automates the prioritization and handling of support tickets submitted via a JotForm form. It is designed to streamline ticket intake, classification, notification, and task creation processes by integrating multiple services:

- **JotForm** for ticket submission trigger.
- **Google Sheets** for logging submitted ticket details.
- Conditional routing based on ticket severity.
- **Slack** notifications for tickets above low severity.
- **Gmail email notifications** for low severity tickets.
- **AI processing** using Google Gemini and Langchain AI Agent to generate concise task descriptions.
- **ClickUp** task creation with AI-generated summaries for efficient project management.

The workflow logic is grouped into these blocks:

- **1.1 Input Reception and Logging**: Receives form submissions and logs data into Google Sheets.
- **1.2 Severity-Based Routing**: Routes tickets to Slack or Gmail based on severity.
- **1.3 AI Processing**: Uses AI to summarize ticket details into a task description.
- **1.4 Task Creation**: Creates a ClickUp task with the AI-generated summary.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Logging

**Overview:**  
This block receives support ticket submissions via JotForm and logs all submitted details into a Google Sheet for record keeping.

**Nodes Involved:**  
- JotForm Trigger  
- Append row in sheet

**Node Details:**

- **JotForm Trigger**  
  - **Type:** Trigger node for JotForm form submissions  
  - **Configuration:** Listens to form ID `252875809016060` for new submissions.  
  - **Inputs:** Incoming webhook/event when a form is submitted.  
  - **Outputs:** JSON containing submitted form data fields (e.g., name, email, severity, issue, description).  
  - **Potential Failures:** Webhook downtime, form ID mismatch, data resolution issues.  
  - **Notes:** Sticky Note: "JotForm Trigger on Form Submission"  

- **Append row in sheet**  
  - **Type:** Google Sheets node  
  - **Configuration:** Appends a new row in the Google Sheet with ID `1hF3MboFa_BppnGPXNs6cjHpGgoRPlYOlnDSAlofsKyg`, sheet gid=0. Columns mapped to form data fields: Name, Email, Issue, Severity, Description.  
  - **Inputs:** Data from JotForm Trigger.  
  - **Outputs:** Confirmation of row append operation.  
  - **Potential Failures:** Google Sheets API auth errors, quota exceeded, schema mismatch.  
  - **Notes:** Sticky Note: "Add the details in Google Sheet"  

---

#### 1.2 Severity-Based Routing

**Overview:**  
This block evaluates the severity of the support ticket and routes the notification either to Slack (for non-low severity) or sends an email (for low severity).

**Nodes Involved:**  
- If  
- Slack  
- Send a message (Gmail)

**Node Details:**

- **If**  
  - **Type:** Conditional routing node  
  - **Configuration:** Checks if the severity field from the form submission is **not equal to "Low"**.  
  - **Inputs:** Output from "Append row in sheet" node (passing along form data).  
  - **Outputs:** Two branches:  
    - True (severity not "Low") → Slack node  
    - False (severity is "Low") → Send a message node  
  - **Potential Failures:** Expression evaluation errors if severity field missing or malformed.  
  - **Notes:** Sticky Note on this and routing nodes: "If low severity then email otherwise slack"  

- **Slack**  
  - **Type:** Slack node for sending messages  
  - **Configuration:** Sends a formatted message to Slack channel ID `C083NTTSZ7F` with ticket details extracted from the JotForm Trigger data (email, name, severity, problem, description). Uses OAuth2 authentication.  
  - **Inputs:** True output branch of If node.  
  - **Outputs:** Passes data to AI Agent node.  
  - **Potential Failures:** Slack API auth issues, channel not found, rate limits.  

- **Send a message (Gmail)**  
  - **Type:** Gmail node for sending emails  
  - **Configuration:** Sends an email to the ticket submitter's email address with the ticket details. Email content is plain text including email, name, severity, problem, and description. Subject line is "New Support Ticket". Uses OAuth2 for Gmail authentication.  
  - **Inputs:** False output branch of If node.  
  - **Outputs:** Terminal node for low severity tickets.  
  - **Potential Failures:** Gmail API auth errors, rate limits, invalid email addresses.  

---

#### 1.3 AI Processing

**Overview:**  
This block processes ticket details using Google Gemini Chat Model and Langchain AI Agent to generate a concise, professional task description for project management.

**Nodes Involved:**  
- Google Gemini Chat Model  
- AI Agent

**Node Details:**

- **Google Gemini Chat Model**  
  - **Type:** AI Language Model node using Google Gemini chat  
  - **Configuration:** Default options, no parameters specified.  
  - **Inputs:** None directly from the workflow; this node connects as an AI language model resource for the AI Agent node.  
  - **Outputs:** Feeds processed AI output to AI Agent node as language model.  
  - **Potential Failures:** API auth, quota limits, model unavailability.  

- **AI Agent**  
  - **Type:** Langchain AI Agent node  
  - **Configuration:** Receives ticket details from Slack node (only non-low severity tickets). The prompt instructs the AI to act as a project manager and summarize the ticket information into a clear, concise task description formatted for ClickUp, explicitly omitting any preamble text.  
  - **Inputs:** Output from Slack node (ticket data) and AI language model from Google Gemini Chat Model.  
  - **Outputs:** JSON with AI-generated task description text.  
  - **Potential Failures:** AI model response failure, prompt formatting issues.  
  - **Notes:** Sticky Note: "AI Agent to summarize the details to be added in clickup"  
  - **Version-specific:** Uses Langchain agent version 2.2  

---

#### 1.4 Task Creation

**Overview:**  
This block creates a new ClickUp task in a specified team and list, using the AI-generated summary as the task content.

**Nodes Involved:**  
- Create a task (ClickUp)

**Node Details:**

- **Create a task**  
  - **Type:** ClickUp node  
  - **Configuration:** Creates a task in ClickUp with:  
    - Team ID: `36299117`  
    - Space ID: `66018999`  
    - Folder ID: `90090948711`  
    - List ID: `900901763830`  
    - Task name: The issue title from the form (`q6_issue`)  
    - Task content: AI Agent output text summarizing the issue.  
  - **Inputs:** AI Agent node output.  
  - **Outputs:** Confirmation of task creation.  
  - **Potential Failures:** ClickUp API auth, invalid IDs, rate limits.  
  - **Notes:** Sticky Note: "Create a task on clickup"  

---

### 3. Summary Table

| Node Name           | Node Type                    | Functional Role                           | Input Node(s)         | Output Node(s)           | Sticky Note                                       |
|---------------------|------------------------------|-----------------------------------------|-----------------------|--------------------------|--------------------------------------------------|
| JotForm Trigger     | jotFormTrigger                | Receive support ticket submissions      | (trigger)             | Append row in sheet       | JotForm Trigger on Form Submission                |
| Append row in sheet | googleSheets                 | Log ticket details into Google Sheets   | JotForm Trigger       | If                       | Add the details in Google Sheet                    |
| If                  | if                          | Route based on severity                  | Append row in sheet   | Slack, Send a message     | If low severity then email otherwise slack        |
| Slack                | slack                        | Notify team via Slack for high severity | If                    | AI Agent                 | If low severity then email otherwise slack        |
| Send a message       | gmail                        | Email user for low severity tickets     | If                    | (end)                    | If low severity then email otherwise slack        |
| Google Gemini Chat Model | lmChatGoogleGemini         | AI language model resource               | (standalone)           | AI Agent                 | AI Model                                           |
| AI Agent             | langchain.agent              | Generate AI summary of ticket            | Slack, Google Gemini   | Create a task             | AI Agent to summarize the details to be added in clickup |
| Create a task        | clickUp                      | Create ClickUp task with AI summary      | AI Agent               | (end)                    | Create a task on clickup                           |
| Sticky Note          | stickyNote                   | Visual notes                            |                       |                          | JotForm Trigger on Form Submission                |
| Sticky Note1         | stickyNote                   | Visual notes                            |                       |                          | Add the details in Google Sheet                    |
| Sticky Note2         | stickyNote                   | Visual notes                            |                       |                          | If low severity then email otherwise slack        |
| Sticky Note3         | stickyNote                   | Visual notes                            |                       |                          | If low severity then email otherwise slack        |
| Sticky Note4         | stickyNote                   | Visual notes                            |                       |                          | AI Model                                           |
| Sticky Note5         | stickyNote                   | Visual notes                            |                       |                          | Create a task on clickup                           |
| Sticky Note6         | stickyNote                   | Visual notes                            |                       |                          | AI Agent to summarize the details to be added in clickup |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add JotForm Trigger node:**  
   - Node Type: `JotForm Trigger`  
   - Configure with your JotForm account.  
   - Set the form ID to `252875809016060` (replace with your form ID).  
   - Enable webhook to listen for new submissions.

3. **Add Google Sheets node to append a row:**  
   - Node Type: `Google Sheets`  
   - Operation: Append  
   - Connect to your Google Sheets account.  
   - Document ID: Use your Google Sheet ID where you want to log tickets.  
   - Sheet Name: Select the correct sheet (gid=0 or by name).  
   - Map columns:  
     - Name → `{{ $json.q3_name.first }}`  
     - Email → `{{ $json.q4_email }}`  
     - Issue → `{{ $json.q6_issue }}`  
     - Severity → `{{ $json.q5_severity }}`  
     - Description → `{{ $json.q7_description }}`  
   - Connect input from JotForm Trigger.

4. **Add If node to route by severity:**  
   - Node Type: `If`  
   - Condition:  
     - Expression: `{{ $json.Severity[0] !== "Low" }}` or use the condition: Severity not equals "Low".  
   - Connect input from Google Sheets append node.

5. **Add Slack node for non-low severity:**  
   - Node Type: `Slack`  
   - Connect your Slack OAuth2 credentials.  
   - Channel: `C083NTTSZ7F` (replace with your channel ID).  
   - Text (message): Use expressions to include:  
     ```
     Email: {{$json.q4_email}}
     Name: {{$json.q3_name.first}} {{$json.q3_name.last}}
     Severity: {{$json.q5_severity}}

     Problem: {{$json.q6_issue}}

     Description: {{$json.q7_description}}
     ```  
   - Connect input from If node True output.

6. **Add Gmail node to send email for low severity:**  
   - Node Type: `Gmail`  
   - Connect your Gmail OAuth2 credentials.  
   - Send To: `={{ $json.q4_email }}`  
   - Subject: "New Support Ticket"  
   - Message (plain text): Include same details as Slack message using expressions.  
   - Connect input from If node False output.

7. **Add Google Gemini Chat Model node:**  
   - Node Type: `lmChatGoogleGemini`  
   - No special parameters required for default setup.  
   - This node is used as a language model resource.

8. **Add AI Agent (Langchain) node:**  
   - Node Type: `Langchain Agent`  
   - Configure prompt to instruct the AI:  
     ```
     You are a project manager. Based on the following support ticket details, create a clear and professional task description that summarizes the issue and highlights key points.

     Ticket Details:

     Name: {{ $json.q3_name.first }} {{ $json.q3_name.last }}

     Severity: {{ $json.q5_severity }}

     Issue: {{ $json.q6_issue }}

     Description: {{ $json.q7_description }}

     Output Format:
     Return a concise task description suitable for a ClickUp task.

     don't add this:
     Here's a concise and professional task description for ClickUp:
     just need the details for clickup
     ```
   - Connect two inputs:  
     - The main input from Slack node output.  
     - The AI language model input from Google Gemini Chat Model node.

9. **Add ClickUp node to create a task:**  
   - Node Type: `ClickUp`  
   - Connect your ClickUp OAuth2 credentials.  
   - Configure:  
     - Team: `36299117`  
     - Space: `66018999`  
     - Folder: `90090948711`  
     - List: `900901763830`  
     - Task name: `={{ $json.q6_issue }}`  
     - Additional field "content": `={{ $json.output }}` (from AI Agent)  
   - Connect input from AI Agent node.

10. **Connect all nodes in sequence:**  
    - JotForm Trigger → Append row in sheet → If →  
      - True → Slack → AI Agent → Create a task  
      - False → Send a message (Gmail)  

11. **Test the workflow:**  
    - Submit a test support ticket via your JotForm form.  
    - Confirm row added in Google Sheets.  
    - For low severity, confirm email sent.  
    - For higher severity, confirm Slack notification, AI summary generation, and ClickUp task creation.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                           |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow uses OAuth2 authentication for Slack, Gmail, and ClickUp integrations.          | Ensure OAuth2 credentials are properly configured with required scopes. |
| Google Gemini Chat Model is used as the AI language model behind the Langchain AI Agent.     | Requires Google Cloud AI API access and billing setup.    |
| Form ID, Slack channel ID, Google Sheet ID, and ClickUp IDs are hardcoded and must be updated to match your environment. | Replace IDs with your actual resources when reproducing workflow. |
| Sticky notes in the workflow provide contextual guidance on each block for ease of maintenance. | Visual aids inside n8n editor.                             |
| This workflow can be extended to add error-handling nodes such as "Error Trigger" or notifications on failures. | Recommended for production deployments.                    |

---

**Disclaimer:**  
The text provided is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.