Automate Multi-step Onboarding with Google Sheets, Forms and Gmail Notifications

https://n8nworkflows.xyz/workflows/automate-multi-step-onboarding-with-google-sheets--forms-and-gmail-notifications-7809


# Automate Multi-step Onboarding with Google Sheets, Forms and Gmail Notifications

### 1. Workflow Overview

This n8n workflow automates a multi-step onboarding process leveraging Google Sheets for data storage, Google Forms for data collection, and Gmail for email notifications. It is designed for HR or administrative teams to manage and track employee onboarding steps efficiently, sending standardized messages to users and administrators based on onboarding progress and status.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Collects onboarding registration data via a form and saves it to a Google Sheet.
- **1.2 Data Retrieval and Preparation:** Reads standardized messages and current onboarding statuses from Google Sheets, generating a message structure for further processing.
- **1.3 Step Processing & Conditional Routing:** Evaluates the current onboarding step and status, then routes the logic to handle different cases like sending messages or updating statuses.
- **1.4 Message Generation and Sending:** Prepares email content by injecting variables into templates and sends emails to users and admins via Gmail and sub-workflows.
- **1.5 Status Update and Next Step Creation:** Updates the onboarding step status in Google Sheets and creates the next onboarding step record if applicable.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block handles the initial onboarding data input from a user via a Google Form and stores the submitted registration data into a Google Sheet.

**Nodes Involved:**  
- Registration (Form Trigger)  
- Save registration Data (Google Sheets Append)  
- Form (Form Completion Acknowledgment)

**Node Details:**  

- **Registration**  
  - Type: Form Trigger  
  - Role: Entry point that triggers when a user submits the onboarding form.  
  - Configuration: Form titled "Welcome to our company" with required fields: First Name, Last Name, Email, plus hidden fields for step (default 1) and status (default "sent").  
  - Input: Form submission data  
  - Output: Triggers "Save registration Data"

- **Save registration Data**  
  - Type: Google Sheets (Append)  
  - Role: Saves submitted form data into the Google Sheet "Standardized messages status" (gid=0).  
  - Configuration: Auto-maps input data to columns like First Name, Last Name, Email, step, status, etc.  
  - Credentials: Google Sheets OAuth2  
  - Input: Data from Registration node  
  - Output: Triggers "Form" node

- **Form**  
  - Type: Form node (Completion)  
  - Role: Sends a completion message "Thanks a lot for entering your information" to the form submitter.  
  - Input: Triggered after data is saved  
  - Output: Ends this input reception flow

**Edge Cases / Failures:**  
- Form submission without required fields will be rejected by the form itself.  
- Google Sheets append may fail if credentials expire or sheet permissions change.  
- Network issues may cause delays or failures.

---

#### 1.2 Data Retrieval and Preparation

**Overview:**  
Reads the standardized messages and the current onboarding status rows from Google Sheets, then processes the message data into a structured table for use in later steps.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Read Messages (Google Sheets Read)  
- Generate message table (Code)  
- Read MSG status Rows (Google Sheets Read)  
- Set variables (Set)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually for testing or batch processing  
  - Output: Triggers "Read Messages"

- **Read Messages**  
  - Type: Google Sheets (Read)  
  - Role: Reads message templates and standardized message content from a Google Sheet "Standardized Messages" (gid=0), used for email content.  
  - Credentials: Google Sheets OAuth2  
  - Input: Triggered by manual trigger  
  - Output: Triggers "Generate message table"

- **Generate message table**  
  - Type: Code (JavaScript)  
  - Role: Processes all message rows into a nested object keyed by step and message type (e.g., sent, completed, error), grouping their content for easy reference.  
  - Code snippet: Groups input items by step and Type, aggregating remaining fields.  
  - Input: Data from "Read Messages"  
  - Output: Triggers "Read MSG status Rows"

- **Read MSG status Rows**  
  - Type: Google Sheets (Read)  
  - Role: Reads current onboarding status entries from the sheet "Standardized messages status" (gid=0), which tracks each user’s onboarding step and status.  
  - Credentials: Google Sheets OAuth2  
  - Input: From "Generate message table"  
  - Output: Triggers "Set variables"

- **Set variables**  
  - Type: Set  
  - Role: Defines variables for message template replacement, e.g., constructs an email address from first and last name, and sets an error message string.  
  - Uses expressions to build an object with keys like `email` and `error`.  
  - Input: From "Read MSG status Rows"  
  - Output: Triggers "Switch type of step"

**Edge Cases / Failures:**  
- Google Sheets read errors due to permission, network, or quota issues.  
- Code node expression failures if input data is malformed or missing expected keys.

---

#### 1.3 Step Processing & Conditional Routing

**Overview:**  
Determines the current step and status of each onboarding entry and routes the workflow to the appropriate handling logic, including sending messages or updating statuses.

**Nodes Involved:**  
- Switch type of step (Switch)  
- Switch (Switch)  
- Switch1 (Switch)  
- Do some action (NoOp)  
- Check if action done (NoOp)  
- Debug mode set result (Set)  
- If completed4 (If)

**Node Details:**  

- **Switch type of step**  
  - Type: Switch  
  - Role: Routes based on the status field of each onboarding entry ("sent" or "completed").  
  - Expression checks status field of current row.  
  - Outputs:  
    - "sent" routes to "Switch" node  
    - "completed" routes to "Switch1" node

- **Switch** and **Switch1**  
  - Type: Switch  
  - Role: Each routes based on the onboarding step number (1 through 5).  
  - Conditions check if step equals 1, 2, 3, 4, or 5, renaming outputs accordingly.  
  - Output: Both connect to "Do some action" and "Check if action done" respectively.

- **Do some action**  
  - Type: NoOp  
  - Role: Placeholder for step-specific actions; in this workflow forwards to sending messages.  
  - Input: Routed from "Switch" for "sent" status steps.  
  - Output: Triggers "Send msg to user"

- **Check if action done**  
  - Type: NoOp  
  - Role: Placeholder for verification after completion; routes to set debug flag.  
  - Input: Routed from "Switch1" (completed steps)  
  - Output: Triggers "Debug mode set result"

- **Debug mode set result**  
  - Type: Set  
  - Role: Sets a boolean variable `check` to true, used to control subsequent conditional flow.  
  - Input: From "Check if action done"  
  - Output: Triggers "If completed4"

- **If completed4**  
  - Type: If  
  - Role: Branches flow depending on the `check` boolean variable.  
  - Condition: `check` is true  
  - True branch: triggers "Send msg to user1" (completed message)  
  - False branch: triggers "Send msg to admin1" (error notification)

**Edge Cases / Failures:**  
- If the status or step data is missing or malformed, nodes may not route correctly.  
- NoOp nodes do not produce errors but represent placeholders that could be extended.  
- Boolean condition failures or expression errors in If node.

---

#### 1.4 Message Generation and Sending

**Overview:**  
Prepares the email message content by substituting variables into templates, then sends the emails using Gmail or sub-workflows dedicated to sending messages to users or administrators.

**Nodes Involved:**  
- Send msg to user (Execute Workflow)  
- Send msg to admin (Execute Workflow)  
- Send msg to user1 (Execute Workflow)  
- Send msg to admin1 (Execute Workflow)  
- Send message (Execute Workflow Trigger)  
- Code (Code)  
- Gmail2 (Gmail Send)

**Node Details:**  

- **Send msg to user**, **Send msg to admin**, **Send msg to user1**, **Send msg to admin1**  
  - Type: Execute Workflow  
  - Role: Calls a sub-workflow (ID: EORpbQwSc80AYgSs) that sends the email messages.  
  - Inputs include: recipient email, subject, email content, and variables for template substitution.  
  - Mapping passes dynamic data from the main workflow’s processed message table and variables set earlier.  
  - "Send msg to user" handles messages for "sent" status steps.  
  - "Send msg to user1" and "Send msg to admin1" handle completed or error messages.

- **Send message**  
  - Type: Execute Workflow Trigger  
  - Role: Triggers the sub-workflow responsible for sending messages with inputs email, subject, content, and variables.

- **Code**  
  - Type: Code Node  
  - Role: Replaces template variables enclosed in `{{ }}` with actual values from the variables object.  
  - Input: Receives content template and variables object.  
  - Output: Returns the final `msg` string ready for sending.

- **Gmail2**  
  - Type: Gmail  
  - Role: Sends the actual email using Gmail OAuth2 credentials.  
  - Uses expressions to fill "To", "Subject", and "Message" from processed data.  
  - Configured with a dedicated Gmail account credential.  
  - Triggered by the "Code" node output.

**Edge Cases / Failures:**  
- Sub-workflow failures or misconfiguration may prevent messaging.  
- Template variables not found in substitution may leave raw placeholders in messages.  
- Gmail API quota or authentication failures.  
- Network or rate limits from Gmail.

---

#### 1.5 Status Update and Next Step Creation

**Overview:**  
Updates the Google Sheet to reflect message sending progress by changing the status to "completed" or "passed" and creates the next step entry for the user in the onboarding process.

**Nodes Involved:**  
- Update row to completed (Google Sheets Update)  
- Update row to passed (Google Sheets Update)  
- Create next step (Google Sheets Append)

**Node Details:**  

- **Update row to completed**  
  - Type: Google Sheets (Update)  
  - Role: Updates the current onboarding row’s status to "completed" after sending a message for "sent" status step.  
  - Uses row_number to identify the row to update.  
  - Credentials: Google Sheets OAuth2 (new project credentials)  
  - Input: From "Send msg to user"

- **Update row to passed**  
  - Type: Google Sheets (Update)  
  - Role: Updates the onboarding status to "passed" after successful completion messages are sent.  
  - Uses row_number to identify row.  
  - Credentials: Google Sheets OAuth2  
  - Input: From "Send msg to user1"  
  - Output: Triggers "Create next step"

- **Create next step**  
  - Type: Google Sheets (Append)  
  - Role: Adds a new row to the status sheet representing the next onboarding step (step number incremented by 1, status set to "sent") for the user.  
  - Inputs use expressions to carry forward First Name, Last Name, Email, and increment step.  
  - Credentials: Google Sheets OAuth2  
  - Input: From "Update row to passed"

**Edge Cases / Failures:**  
- Row update failures due to incorrect row_number or concurrency issues.  
- Append failures if Google Sheets quota is exceeded or permissions revoked.  
- Logic may fail if step number is missing or not numeric.

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                           | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                   |
|---------------------------|----------------------------|-----------------------------------------|-----------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Manual start of message reading          | -                                 | Read Messages                    |                                                                                               |
| Generate message table     | Code                       | Processes messages into indexed table    | Read Messages                     | Read MSG status Rows             |                                                                                               |
| Read MSG status Rows       | Google Sheets              | Reads current onboarding statuses        | Generate message table            | Set variables                   |                                                                                               |
| Schedule Trigger           | Schedule Trigger (disabled)| Scheduled start (disabled)                 | -                                 | Read Messages                   |                                                                                               |
| If completed4              | If                         | Conditional branch on completion check   | Debug mode set result             | Send msg to user1, Send msg to admin1 |                                                                                               |
| Form                      | Form                       | Collects registration data via form      | Save registration Data            | -                              |                                                                                               |
| Save registration Data     | Google Sheets              | Appends registration data to sheet       | Registration                     | Form                           |                                                                                               |
| Read Messages             | Google Sheets              | Reads standardized messages from sheet   | When clicking ‘Execute workflow’ | Generate message table          |                                                                                               |
| Update row to completed    | Google Sheets              | Updates user status to completed          | Send msg to user                 | -                              |                                                                                               |
| Do some action            | NoOp                       | Placeholder for step actions              | Switch                          | Send msg to user               |                                                                                               |
| Check if action done       | NoOp                       | Placeholder for verification              | Switch1                         | Debug mode set result           |                                                                                               |
| Create next step           | Google Sheets              | Appends next onboarding step              | Update row to passed             | -                              |                                                                                               |
| Send message              | Execute Workflow Trigger   | Triggers sub-workflow for message sending | -                               | Code                           |                                                                                               |
| Gmail2                   | Gmail                      | Sends email via Gmail                      | Code                           | -                              |                                                                                               |
| Send msg to user           | Execute Workflow           | Sends message to user                      | Do some action                  | Update row to completed, Send msg to admin |                                                                                               |
| Send msg to admin          | Execute Workflow           | Sends error message to admin               | Send msg to user                | -                              |                                                                                               |
| Send msg to user1          | Execute Workflow           | Sends completed message to user            | If completed4                   | Update row to passed           |                                                                                               |
| Send msg to admin1         | Execute Workflow           | Sends error message to admin on failure    | If completed4                   | -                              |                                                                                               |
| Set variables             | Set                        | Defines variables for message templates   | Read MSG status Rows             | Switch type of step            |                                                                                               |
| Sticky Note               | Sticky Note                | Explains starting point                    | -                               | -                              | ## This is the starting point\n- You can send the link by email to the person\n- Once it will fill it it will create the Step 1 |
| Switch                   | Switch                     | Routes based on step for "sent" status     | Switch type of step             | Do some action                 | ## Case setting \nHere you can set the distinct action for each step                          |
| Switch1                  | Switch                     | Routes based on step for "completed" status | Switch type of step             | Check if action done           | ## Case setting \nHere you can set the distinct verifications for each step                   |
| Sticky Note1              | Sticky Note                | Describes case setting                      | -                               | -                              | ## Case setting \nHere you can set the distinct action for each step                          |
| Sticky Note2              | Sticky Note                | Describes case setting                      | -                               | -                              | ## Case setting \nHere you can set the distinct verifications for each step                   |
| Sticky Note9              | Sticky Note                | Marks main workflow                          | -                               | -                              | ## Main workflow                                                                              |
| Switch type of step       | Switch                     | Routes based on status ("sent" or "completed") | Set variables                  | Switch, Switch1               |                                                                                               |
| Debug mode set result     | Set                        | Sets boolean check flag                      | Check if action done            | If completed4                 |                                                                                               |
| Sticky Note3              | Sticky Note                | Notes about sub-workflow message sending   | -                               | -                              | ## Sub Workflow \n- replace the variables enclosed in {{ }} by its corresponding value set in "Set variables node".\n- Send messages, here you can set a slack or telegram message |
| Update row to passed      | Google Sheets              | Updates status to "passed"                   | Send msg to user1               | Create next step             |                                                                                               |
| Registration             | Form Trigger               | Triggers on form submission                  | -                               | Save registration Data        |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Form for Registration:**
   - Use the "Form Trigger" node.
   - Title: "Welcome to our company".
   - Add required fields: First Name, Last Name, Email.
   - Add hidden fields: step (default `1`), status (default `"sent"`).
   - Path: `/onboarding`.

2. **Save Registration Data:**
   - Add a Google Sheets node configured to append data.
   - Use a Google Sheet document to store registration data.
   - Map form fields to columns accordingly.
   - Connect Form Trigger output to this node.

3. **Send Completion Message:**
   - Add a Form node with operation "completion".
   - Set completion title: "Thanks a lot for entering your information".
   - Connect from Google Sheets append node.

4. **Manual Trigger to Start Batch Process:**
   - Add Manual Trigger node.
   - Connect its output to a Google Sheets node reading standardized messages.
   - Use a separate Google Sheet designed as a message template repository.
   - Authenticate with Google Sheets OAuth2.

5. **Process Messages:**
   - Add a Code node that groups messages by step and type.
   - Use JavaScript code to build a nested object keyed by step and message type.

6. **Read Onboarding Status Rows:**
   - Add a Google Sheets read node.
   - Point to the onboarding status sheet.
   - Authenticate with Google Sheets OAuth2.
   - Connect after message processing.

7. **Set Variables for Message Templates:**
   - Add a Set node.
   - Create an object `variables` with keys like `email` (constructed from First and Last names) and `error` message.

8. **Switch Node for Status Routing:**
   - Add a Switch node evaluating the `status` field.
   - Routes "sent" to one branch, "completed" to another.

9. **Step-Based Switches:**
   - Add two Switch nodes for each branch of status.
   - Configure rules for steps 1 through 5.

10. **Placeholder Action Nodes:**
    - Add NoOp nodes for "Do some action" and "Check if action done".
    - Connect step switches to these.

11. **Set Debug Flag:**
    - Add a Set node to assign `check = true`.
    - Connect from "Check if action done" NoOp.

12. **Conditional Branch:**
    - Add an If node checking if `check` is true.
    - True branch: sends completed message.
    - False branch: sends error message to admin.

13. **Send Messages via Sub-Workflows:**
    - Add Execute Workflow nodes calling a sub-workflow for sending emails.
    - Pass parameters: email, subject, content, variables.
    - Use expressions to select message content from the generated message table.

14. **Message Content Preparation:**
    - Add Execute Workflow Trigger node to trigger sub-workflow.
    - Add Code node that replaces template variables in the message content.
    - Add Gmail node configured with Gmail OAuth2 to send emails.
    - Link Code node output to Gmail node input.

15. **Update Status Rows:**
    - Add Google Sheets update nodes to mark rows as "completed" or "passed".
    - Use the row_number from sheets to identify rows.
    - Connect after sending messages.

16. **Create Next Step Entry:**
    - Add Google Sheets append node.
    - Append a new row with step incremented by 1 and status "sent".
    - Connect from the "Update row to passed" node.

17. **Credentials:**
    - Configure Google Sheets OAuth2 credentials for all Google Sheets nodes.
    - Configure Gmail OAuth2 credentials for sending emails.
    - Ensure correct access rights for all used Google Sheets.

18. **Testing & Debugging:**
    - Use Manual Trigger to start workflow for testing.
    - Validate form submission flow separately.
    - Monitor logs for errors, especially in variable substitution and Google Sheets updates.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| **Starting point**: You can send the onboarding form link via email; once filled, it creates Step 1 automatically. | Sticky Note: "## This is the starting point\n- You can send the link by email to the person\n- Once it will fill it it will create the Step 1" |
| **Case settings**: Different actions and verifications can be set for each step via switches.                    | Sticky Notes near Switch and Switch1 nodes describing case setting. |
| **Sub-workflow message sending**: Variables enclosed in `{{ }}` are replaced with values set in "Set variables". Messages can also be sent via Slack or Telegram by extending the sub-workflow. | Sticky Note near Send message nodes.                             |
| Project uses multiple Google Sheets for status tracking and message templates to allow easy updates without code changes. | Google Sheets URLs are embedded in node parameters.             |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow built with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.