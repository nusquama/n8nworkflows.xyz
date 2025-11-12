Automatic Invoice Detection & Reminder Creation with Gmail & Google Tasks

https://n8nworkflows.xyz/workflows/automatic-invoice-detection---reminder-creation-with-gmail---google-tasks-5863


# Automatic Invoice Detection & Reminder Creation with Gmail & Google Tasks

### 1. Workflow Overview

This workflow automates the detection of invoice-related emails from a Gmail inbox and creates corresponding tasks in Google Tasks to remind users about payment deadlines. It is designed for users who want to keep track of invoices automatically without manual email triaging.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger**: Periodically triggers the workflow to check for new unread emails.
- **1.2 Email Retrieval**: Fetches unread emails from Gmail.
- **1.3 AI Processing**: Uses an AI agent (LangChain with OpenAI GPT-4o) to analyze email content, determine if the email is invoice-related, and extract key invoice details (due date, amount).
- **1.4 Conditional Processing**: Decides whether the email relates to an invoice.
- **1.5 Invoice Task Creation and Email Labeling**: For invoice emails, adds a label in Gmail, marks the email as read, and creates a Google Task with invoice details.
- **1.6 No Operation**: If email is not invoice-related, performs no action.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow at hourly intervals to check for new unread emails.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Runs every 1 hour (hour interval)  
    - Inputs: None (trigger node)  
    - Outputs: Triggers "Get Unread Emails" node  
    - Edge Cases: Misconfiguration of interval could cause too frequent or infrequent triggers; time zone considerations apply.  
    - Notes: Adjustable interval if user wants to check email more or less frequently.

#### 1.2 Email Retrieval

- **Overview:**  
  Fetches all unread emails from the connected Gmail account with full email details for analysis.

- **Nodes Involved:**  
  - Get Unread Emails

- **Node Details:**  
  - **Get Unread Emails**  
    - Type: Gmail node (getAll operation)  
    - Configuration: Filters emails with "unread" status; fetches complete email data (not simple)  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Provides array of unread emails to "AI Agent"  
    - Edge Cases: Gmail API rate limits, OAuth2 token expiration, no unread emails (empty output)  
    - Failure Modes: Authentication errors, network issues

#### 1.3 AI Processing

- **Overview:**  
  Applies AI to each email's content to identify invoice-related messages and extract structured invoice data.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent node  
    - Configuration:  
      - Uses a prompt instructing the AI to:  
        1. Detect if the email is invoice-related  
        2. Extract due date (YYYY-MM-DD or null)  
        3. Extract amount due (number or null)  
      - Returns only a JSON object with keys: `is_invoice`, `due_date`, `amount_due`, `email_id`, `thread_id`, `sender`, `subject`  
      - On error: continues regular output (does not fail workflow)  
    - Inputs: Emails from "Get Unread Emails"  
    - Outputs: Parsed JSON to "Check If Email is Invoice"  
    - Expressions: Includes email metadata in prompt dynamically (e.g., `{{ $json.id }}`, `{{ $json.text }}`)  
    - Edge Cases: AI misclassification, prompt parsing errors, API rate limits, timeout, malformed JSON output  
  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI GPT model node  
    - Configuration: Uses GPT-4o model  
    - Inputs: Connected as AI language model for "AI Agent"  
    - Outputs: AI-generated content for parsing  
    - Edge Cases: API key invalid, quota exceeded, latency  
  - **Structured Output Parser**  
    - Type: LangChain Output Parser node  
    - Configuration: Validates AI output against a manual schema (not fully shown in JSON)  
    - Inputs: AI Agent output  
    - Outputs: Clean JSON for downstream nodes  
    - Edge Cases: Parsing errors if AI output deviates from schema

#### 1.4 Conditional Processing

- **Overview:**  
  Checks if the AI identified the email as invoice-related to branch the workflow accordingly.

- **Nodes Involved:**  
  - Check If Email is Invoice (IF node)

- **Node Details:**  
  - **Check If Email is Invoice**  
    - Type: IF node  
    - Configuration: Condition checks if `is_invoice` field in AI output is `true` (boolean true)  
    - Inputs: AI Agent output  
    - Outputs:  
      - True branch: proceeds to label email and create task  
      - False branch: routes to No Operation node  
    - Edge Cases: Missing or malformed `is_invoice` field; fallback behavior to prevent workflow failure

#### 1.5 Invoice Task Creation and Email Labeling

- **Overview:**  
  For detected invoice emails, adds a Gmail label "Invoice," marks email as read, and creates a Google Task reminder with invoice details.

- **Nodes Involved:**  
  - Add Label to Email  
  - Mark Email as read  
  - Create a task

- **Node Details:**  
  - **Add Label to Email**  
    - Type: Gmail node  
    - Configuration: Adds the label "Invoice" to the email specified by `email_id` from AI output  
    - Inputs: True branch of IF node  
    - Outputs: Triggers "Mark Email as read"  
    - Edge Cases: Label "Invoice" must exist in Gmail beforehand; label creation not handled here; errors if label missing  
  - **Mark Email as read**  
    - Type: Gmail node  
    - Configuration: Marks the email as read using the email `id`  
    - Inputs: From "Add Label to Email"  
    - Outputs: None (end of chain here)  
    - Edge Cases: Email may be already marked read, API errors  
  - **Create a task**  
    - Type: Google Tasks node  
    - Configuration:  
      - Title formatted as: `Invoice from {{sender}} – ${{amount_due}} due {{due_date}}`  
      - Task description left empty (could be customized)  
    - Inputs: True branch of IF node (parallel to "Add Label to Email")  
    - Outputs: None  
    - Edge Cases: Invalid date or amount formats could affect task readability; Google Tasks API quota or auth issues  
    - Notes: Sticky Note recommends customizing task format and adding more invoice details in notes field.

#### 1.6 No Operation

- **Overview:**  
  Performs no action for emails not identified as invoices, effectively ignoring them.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**  
  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Configuration: Empty placeholder node  
    - Inputs: False branch of IF node  
    - Outputs: None  
    - Edge Cases: None; ensures workflow does not fail or stop on non-invoice emails

---

### 3. Summary Table

| Node Name             | Node Type                        | Functional Role                              | Input Node(s)             | Output Node(s)                   | Sticky Note                                                                                                 |
|-----------------------|---------------------------------|----------------------------------------------|---------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                 | Triggers workflow hourly                      | None                      | Get Unread Emails               | ### 💡 Schedule Trigger  This runs every hour.  You can change the interval here depending on how often you want Gmail to be checked. |
| Get Unread Emails      | Gmail                           | Retrieves all unread emails                    | Schedule Trigger           | AI Agent                       | ### ⚠️ Setup Required  - Connect your Gmail account using OAuth2  - Add your OpenAI API Key under **API Credentials**  - Connect your Google Tasks account  - Create this label in Gmail: **Invoice** |
| AI Agent              | LangChain Agent                 | Determines if email is invoice and extracts data | Get Unread Emails          | Check If Email is Invoice       |                                                                                                             |
| OpenAI Chat Model      | LangChain OpenAI Chat Model     | Provides GPT-4o language model for AI Agent  | AI Agent (ai_languageModel)| AI Agent (ai_languageModel)     |                                                                                                             |
| Structured Output Parser| LangChain Output Parser         | Parses AI JSON output into structured data   | AI Agent (ai_outputParser) | AI Agent (ai_outputParser)      |                                                                                                             |
| Check If Email is Invoice | IF Node                      | Branches workflow based on invoice detection | AI Agent                   | Add Label to Email, Create a task, No Operation |                                                                                                             |
| Add Label to Email     | Gmail                           | Adds "Invoice" label to email                 | Check If Email is Invoice  | Mark Email as read              |                                                                                                             |
| Mark Email as read     | Gmail                           | Marks email as read                            | Add Label to Email         | None                          |                                                                                                             |
| Create a task          | Google Tasks                    | Creates task in Google Tasks with invoice info | Check If Email is Invoice  | None                          | ### 💡 Customize Task Format  You can change the title in this node.  For example: `Pay invoice from {{sender}} by {{due_date}}`  Include more invoice details in the notes field if needed. |
| No Operation, do nothing| NoOp                          | Does nothing for non-invoice emails           | Check If Email is Invoice  | None                          |                                                                                                             |
| Sticky Note            | Sticky Note                     | Informational note for task customization     | None                      | None                          | ### 💡 Customize Task Format  You can change the title in this node.  For example: `Pay invoice from {{sender}} by {{due_date}}`  Include more invoice details in the notes field if needed. |
| Sticky Note1           | Sticky Note                     | Informational note about schedule trigger     | None                      | None                          | ### 💡 Schedule Trigger  This runs every hour.  You can change the interval here depending on how often you want Gmail to be checked. |
| Sticky Note2           | Sticky Note                     | Setup instructions and requirements           | None                      | None                          | ### ⚠️ Setup Required  - Connect your Gmail account using OAuth2  - Add your OpenAI API Key under **API Credentials**  - Connect your Google Tasks account  - Create this label in Gmail: **Invoice** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to run every 1 hour (field: hours)  
   - This node triggers the workflow start.

2. **Add a Gmail node named "Get Unread Emails"**  
   - Operation: Get All  
   - Filters: Read Status = Unread  
   - Set Simple to false to fetch full email content  
   - Connect input to Schedule Trigger node’s output.

3. **Add LangChain AI Agent node named "AI Agent"**  
   - Select LangChain Agent node type  
   - Prompt Type: Define  
   - Prompt text:  
     ```
     You are an intelligent assistant that reads emails and determines whether the message is related to an invoice or a payment notification.

     Your tasks:
     1. Determine if the email is invoice-related.
     2. If yes, extract:
        - Due date (in YYYY-MM-DD format, or null)
        - Amount due (as a number, no currency symbols)

     Always include these metadata values:
     Id: {{ $json.id }}
     threadId: {{ $json.threadId }}
     body: {{ $json.text }}
     subject: {{ $json.subject }}
     sender: {{ $json.from.value[0].name }}

     Return only a valid JSON object in the format below:

     {
       "is_invoice": true or false,
       "due_date": "YYYY-MM-DD" or null,
       "amount_due": number or null,
       "email_id": "string",
       "thread_id": "string",
       "sender": "string",
       "subject": "string"
     }
     ```
   - Link OpenAI Chat Model as AI language model (see next step).  
   - Connect input to "Get Unread Emails" output.

4. **Add LangChain OpenAI Chat Model node named "OpenAI Chat Model"**  
   - Model: GPT-4o  
   - Connect output to AI Agent node as AI language model input.

5. **Add LangChain Structured Output Parser node named "Structured Output Parser"**  
   - Schema Type: Manual  
   - Define JSON schema matching AI Agent output (keys: is_invoice:boolean, due_date:string|null, amount_due:number|null, email_id:string, thread_id:string, sender:string, subject:string)  
   - Connect output to AI Agent node as AI output parser.

6. **Add IF node named "Check If Email is Invoice"**  
   - Condition:  
     - Type: Boolean  
     - Check if `{{ $json.output.is_invoice }}` is true  
   - Connect input to AI Agent output.

7. **Add Gmail node named "Add Label to Email"**  
   - Operation: Add Labels  
   - Label Names: ["Invoice"] (must pre-create this label in Gmail)  
   - Message ID: `{{ $json.output.email_id }}` from AI Agent output  
   - Connect input to IF node’s True output.

8. **Add Gmail node named "Mark Email as read"**  
   - Operation: Mark as Read  
   - Message ID: `{{ $json.id }}` (original email ID)  
   - Connect input from "Add Label to Email" output.

9. **Add Google Tasks node named "Create a task"**  
   - Operation: Create Task  
   - Title: Use expression  
     ```
     Invoice from {{ $json.output.sender }} – ${{ $json.output.amount_due }} due {{ $json.output.due_date }}
     ```  
   - Task description: leave empty or customize as needed  
   - Connect input from IF node’s True output (parallel to "Add Label to Email")

10. **Add No Operation node named "No Operation, do nothing"**  
    - Connect input from IF node’s False output (for emails not identified as invoices)  

11. **Credential Setup:**  
    - Gmail: OAuth2 with proper scopes to read emails, modify labels, mark read  
    - OpenAI: API key configured in n8n credentials  
    - Google Tasks: OAuth2 with access to manage tasks  

12. **Label Setup:**  
    - Create "Invoice" label in Gmail manually before running workflow  

13. **Testing:**  
    - Trigger workflow manually or wait for schedule trigger  
    - Confirm invoice emails are labeled, marked read, and tasks created with correct details  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ⚠️ Setup Required: Connect Gmail (OAuth2), OpenAI API Key, Google Tasks, and create Gmail label "Invoice" | Sticky Note on "Get Unread Emails" and general setup instructions                                |
| 💡 Customize Task Format: Title can be changed; example `Pay invoice from {{sender}} by {{due_date}}`; add more details in task notes | Sticky Note near "Create a task" node                                                           |
| 💡 Schedule Trigger runs every hour by default; interval adjustable depending on desired frequency   | Sticky Note near "Schedule Trigger" node                                                        |
| Workflow uses LangChain integration with OpenAI GPT-4o model for AI processing                        | Requires n8n LangChain nodes and valid OpenAI credentials                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. The processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.