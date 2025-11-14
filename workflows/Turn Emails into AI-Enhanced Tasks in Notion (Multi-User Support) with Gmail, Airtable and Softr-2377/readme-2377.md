Turn Emails into AI-Enhanced Tasks in Notion (Multi-User Support) with Gmail, Airtable and Softr

https://n8nworkflows.xyz/workflows/turn-emails-into-ai-enhanced-tasks-in-notion--multi-user-support--with-gmail--airtable-and-softr-2377


# Turn Emails into AI-Enhanced Tasks in Notion (Multi-User Support) with Gmail, Airtable and Softr

### 1. Workflow Overview

This workflow automates the transformation of forwarded emails into AI-enhanced actionable tasks stored in personal Notion databases. It supports multiple users sharing a single Gmail account and processing pipeline by leveraging a user-configurable routing system maintained in Airtable and exposed via a Softr portal.

The workflow is logically divided into these key blocks:

- **1.1 Input Reception & Filtering**: Listens to incoming emails in Gmail, filters out already processed or error-labeled emails, and extracts routing information from the email alias.

- **1.2 Route Resolution & Validation**: Retrieves route configuration from Airtable based on the extracted route ID and filters out inactive routes.

- **1.3 AI-Driven Email Processing**: Uses OpenAI GPT-4o models to generate an actionable task and a detailed summary/meta data from the email content.

- **1.4 Notion Page Creation**: Formats the AI output into Notion’s API page structure and sends the creation request with user-specific authentication.

- **1.5 Post-Processing & Notifications**: Marks emails as processed or error-labeled accordingly, deactivates faulty routes, and sends notification emails on errors or missing routes.

- **1.6 Initialization & Setup Utilities**: Provides manual triggers and label ID retrieval to configure the workflow’s global parameters.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview**: Monitors Gmail inbox for new emails every minute, skips emails already processed or marked as error, and extracts the routing ID from the email alias.

- **Nodes Involved**: Gmail Trigger, Globals, Filter for unprocessed mails, Extract Route ID

- **Node Details**:

  - **Gmail Trigger**
    - Type: Trigger (Gmail)
    - Config: Watches INBOX label, polls every minute.
    - Input: Incoming new email notifications.
    - Output: Email JSON including labels, to/from fields.
    - Credentials: Gmail OAuth2.
    - Edge cases: Gmail API rate limits; connection errors.
  
  - **Globals**
    - Type: Set node for global variables.
    - Config: Holds Gmail label IDs for “Processed” and “Error” labels.
    - Input: Email data from Gmail Trigger.
    - Output: Passes global label IDs downstream.
    - Setup: Requires manual initialization to fetch label IDs.
  
  - **Filter for unprocessed mails**
    - Type: Filter
    - Config: Filters emails without “Processed” or “Error” labels and with a valid alias extracted from “to” address.
    - Input: Emails from Globals.
    - Output: Only eligible emails for processing pass.
    - Edge cases: Emails missing routing alias; incorrect label detection.
  
  - **Extract Route ID**
    - Type: Set
    - Config: Extracts route ID from email alias using regex capturing the string between "+" and "@".
    - Input: Filtered email JSON.
    - Output: Sets variable `route` with extracted ID.
    - Edge cases: Malformed email addresses causing regex failure.

#### 2.2 Route Resolution & Validation

- **Overview**: Looks up the route configurations in Airtable by route ID and filters to only active routes.

- **Nodes Involved**: Get Route by ID, Active Routes Only, No Operation (do nothing)

- **Node Details**:

  - **Get Route by ID**
    - Type: Airtable (Get operation)
    - Config: Retrieves route record by ID from “Routes” table.
    - Input: Route ID from Extract Route ID node.
    - Output: Route configuration including Notion API token, Database URL, and active status.
    - Credentials: Airtable API key.
    - Error Handling: On error, continues with output.
    - Edge cases: Non-existent route ID, Airtable API outages.
  
  - **Active Routes Only**
    - Type: Filter
    - Config: Passes only if the route’s “Active” checkbox is true.
    - Input: Route data from Get Route by ID.
    - Output: Active routes proceed; inactive routes trigger no-op.
  
  - **No Operation, do nothing**
    - Type: NoOp
    - Config: Pass-through node used for routing inactive routes.
    - Output: Triggers “Send notification about missing route” downstream.

#### 2.3 AI-Driven Email Processing

- **Overview**: Processes email content with OpenAI GPT-4o models to generate a structured actionable task and a detailed summary with metadata.

- **Nodes Involved**: Generate Actionable Task, Structured Output Parser1, Get Summary & Meta Data, Structured Output Parser, OpenAI Chat Model1, OpenAI Chat Model

- **Node Details**:

  - **Generate Actionable Task**
    - Type: LangChain Agent node (AI agent)
    - Config: Prompt instructs AI to extract one actionable task from email text, with title, description, optional bullet points.
    - Input: Raw email text.
    - Output: JSON with task attributes.
    - Output parsing: Structured Output Parser1 parses AI JSON.
    - Edge cases: Ambiguous email content; AI hallucination; API quota limits.
  
  - **Get Summary & Meta Data**
    - Type: LangChain Agent node (AI agent)
    - Config: Summarizes email, extracts sender, subject, date with specified formats.
    - Input: Raw email text.
    - Output: JSON with summary and meta attributes.
    - Output parsing: Structured Output Parser parses AI JSON.
  
  - **OpenAI Chat Model1 and OpenAI Chat Model**
    - Type: OpenAI GPT-4o language model nodes.
    - Role: Provide language model inference for respective AI agent nodes.
    - Configuration: Temperature 0 for deterministic output, JSON response format.
    - Credentials: OpenAI API key.
    - Edge cases: API errors, timeout, rate limits.

#### 2.4 Notion Page Creation

- **Overview**: Formats AI-generated content into the required Notion API page structure and creates a new page in the user’s Notion database with dynamic authentication.

- **Nodes Involved**: Format Notion Page Blocks, Create Notion Page

- **Node Details**:

  - **Format Notion Page Blocks**
    - Type: Code (JavaScript)
    - Config: Combines actionable task and summary/meta data into Notion page blocks including paragraphs, bullet points, dividers.
    - Uses regex to extract Notion Database ID from URL.
    - Output: JSON body for Notion API POST request.
    - Edge cases: Incorrect Notion database URL format; missing AI data.
  
  - **Create Notion Page**
    - Type: HTTP Request
    - Config: Sends POST request to Notion API /v1/pages with dynamic Bearer token from route config.
    - Headers: Authorization via user’s Notion API token; Notion version header set.
    - Error Handling: Retries on failure; on error, continues with error output.
    - Edge cases: Notion API downtime, invalid/not authorized token, malformed request body.

#### 2.5 Post-Processing & Notifications

- **Overview**: Handles labeling emails in Gmail as processed or error, deactivates faulty routes in Airtable, and notifies users by email about errors or missing routes.

- **Nodes Involved**: Add Label "Processed", Add Label "Error", Deactivate Route, Send notification about deactivated route, Send notification about missing route

- **Node Details**:

  - **Add Label "Processed"**
    - Type: Gmail node (Add Labels)
    - Config: Adds “Processed” label to email after successful Notion page creation.
    - Credentials: Gmail OAuth2.
    - Inputs: Email ID from Gmail Trigger, label ID from Globals.
  
  - **Add Label "Error"**
    - Type: Gmail node (Add Labels)
    - Config: Adds “Error” label to email on failure.
  
  - **Deactivate Route**
    - Type: Airtable (Update operation)
    - Config: Sets route’s “Active” checkbox to false to pause route after error.
    - Input: Route ID from Get Route by ID.
    - Credentials: Airtable API key.
  
  - **Send notification about deactivated route**
    - Type: Gmail Send Email
    - Config: Sends error notification email to original sender explaining route deactivation and troubleshooting.
  
  - **Send notification about missing route**
    - Type: Gmail Send Email
    - Config: Sends email to sender if no active route found for their forwarded email alias.

#### 2.6 Initialization & Setup Utilities

- **Overview**: Provides manual trigger and label retrieval to initialize global Gmail label IDs for “Processed” and “Error” labels.

- **Nodes Involved**: When clicking ‘Test workflow’, Get all labels, Required labels, Globals

- **Node Details**:

  - **When clicking ‘Test workflow’**
    - Type: Manual Trigger
    - Disabled by default.
  
  - **Get all labels**
    - Type: Gmail (Label List)
    - Config: Fetches all Gmail labels for the configured Gmail account.
  
  - **Required labels**
    - Type: Filter
    - Config: Filters label list for names containing “Error” or “Processed”.
  
  - **Globals**
    - Type: Set node to store label IDs.
    - Used to manually copy label IDs from filtered output for usage in workflow.

---

### 3. Summary Table

| Node Name                            | Node Type                                 | Functional Role                                    | Input Node(s)                   | Output Node(s)                         | Sticky Note                                                                                                                   |
|------------------------------------|-------------------------------------------|--------------------------------------------------|--------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger                      | Trigger (Gmail)                           | Detect incoming emails                            | -                              | Globals                               | The Gmail inbox is checked every minute for new entries                                                                       |
| Globals                           | Set                                       | Define global Gmail label IDs                     | Gmail Trigger                  | Filter for unprocessed mails           | Use the setup instructions below to retrieve the values for both `errorLabelID` and `processedLabelID`                        |
| Filter for unprocessed mails       | Filter                                    | Skip emails already processed or errored          | Globals                        | Extract Route ID                      | Filter by labels to prohibit double-processing                                                                                |
| Extract Route ID                  | Set                                       | Extract route ID from email alias                 | Filter for unprocessed mails   | Get Route by ID                      | Extract the Airtable Row ID from the Email address                                                                             |
| Get Route by ID                  | Airtable (Get)                            | Retrieve route configuration by ID                | Extract Route ID               | Active Routes Only, No Operation, do nothing | Select the database and the table where the "Routes" are defined                                                               |
| Active Routes Only               | Filter                                    | Pass only active routes                            | Get Route by ID                | Generate Actionable Task               | Skip disabled routes (determined by a checkbox attribute in Airtable)                                                         |
| No Operation, do nothing          | NoOp                                      | Handle inactive routes                             | Get Route by ID                | Send notification about missing route |                                                                                                                              |
| Generate Actionable Task          | LangChain Agent                           | Generate actionable task from email                | Active Routes Only             | Get Summary & Meta Data                | The Email is processed in multiple ways: - An actionable task is created, with title, description, bullet points              |
| Structured Output Parser1          | LangChain Output Parser                   | Parse AI JSON output for actionable task           | Generate Actionable Task       | -                                     |                                                                                                                              |
| Get Summary & Meta Data          | LangChain Agent                           | Generate detailed summary and metadata             | Generate Actionable Task       | Format Notion Page Blocks              | The Email is processed in multiple ways: - Summary and metadata extraction for reference                                        |
| Structured Output Parser          | LangChain Output Parser                   | Parse AI JSON output for summary/meta               | Get Summary & Meta Data        | -                                     |                                                                                                                              |
| OpenAI Chat Model1               | OpenAI GPT-4o Model                       | AI inference for actionable task                   | Generate Actionable Task       | Structured Output Parser1              |                                                                                                                              |
| OpenAI Chat Model                | OpenAI GPT-4o Model                       | AI inference for summary/meta data                 | Get Summary & Meta Data        | Structured Output Parser               |                                                                                                                              |
| Format Notion Page Blocks         | Code                                      | Format AI output into Notion API page structure    | Get Summary & Meta Data        | Create Notion Page                    | Dynamically build request body for Notion, since dynamic auth, and content with optional fields require a custom request      |
| Create Notion Page               | HTTP Request                             | Create Notion page with user-specific token        | Format Notion Page Blocks      | Add Label "Processed", Deactivate Route | The custom built request including the user specific authentication is sent to Notion to create a new Page inside of a database |
| Add Label "Processed"            | Gmail (Add Labels)                       | Label email as processed                           | Create Notion Page            | -                                     | Add labels which prevent from double-processing                                                                               |
| Deactivate Route                | Airtable (Update)                        | Deactivate route after error                        | Create Notion Page            | Send notification about deactivated route | Disable a checkbox attribute in Airtable which determines if a route is active                                                 |
| Send notification about deactivated route | Gmail (Send Email)                        | Notify user of route deactivation due to error     | Deactivate Route             | Add Label "Error"                    | Send custom Email notifications back to sender, containing an error message and suggestions to fix it                          |
| Add Label "Error"               | Gmail (Add Labels)                       | Label email as error                                | Send notification about deactivated route, Send notification about missing route | - | Add labels which prevent from double-processing                                                                               |
| Send notification about missing route | Gmail (Send Email)                        | Notify user no active route exists for alias       | No Operation, do nothing      | Add Label "Error"                    | Send custom Email notifications back to sender, containing an error message and suggestions to fix it                          |
| When clicking ‘Test workflow’    | Manual Trigger                           | Manual start for label ID retrieval                 | -                            | Get all labels                      | Disable the Gmail Trigger and enable the manual trigger here Execute the workflow once Copy label IDs to Globals             |
| Get all labels                  | Gmail (List Labels)                      | Retrieve all Gmail labels                            | When clicking ‘Test workflow’ | Required labels                    |                                                                                                                              |
| Required labels                | Filter                                    | Filter labels for “Processed” and “Error”           | Get all labels                | Globals                           |                                                                                                                              |
| Sticky Note (multiple nodes)    | Sticky Note                              | Setup instructions and explanations                  | -                            | -                                   | Multiple notes explaining setup, labeling, AI processing, and Notion integration (refer to section 5)                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**
   - Type: Gmail Trigger
   - Configure to watch the INBOX label.
   - Set polling interval to every minute.
   - Connect Gmail OAuth2 credentials.
   - Connect output to the Globals node.

2. **Create Globals Set Node**
   - Type: Set
   - Create two string variables: `errorLabelID` and `processedLabelID`.
   - Initially leave values blank; will be set via manual trigger.
   - Connect input from Gmail Trigger.
   - Connect output to Filter for unprocessed mails.

3. **Create Filter for Unprocessed Mails**
   - Type: Filter
   - Conditions:
     - Email label IDs do NOT contain `errorLabelID` or `processedLabelID`.
     - Email “to” address contains alias matching regex `\+([^@]+)@`.
   - Connect input from Globals.
   - Output only passing emails to Extract Route ID.

4. **Create Extract Route ID Set Node**
   - Type: Set
   - Extract `route` variable using expression: `{{$json["to"]["text"].match(/\+([^@]+)@/)[1]}}`
   - Connect input from Filter.
   - Connect output to Get Route by ID.

5. **Create Airtable Get Node (Get Route by ID)**
   - Set operation: Get record by ID.
   - Base and Table: Select your Airtable base and “Routes” table.
   - Use `route` variable as record ID.
   - Connect Airtable API credentials.
   - Connect input from Extract Route ID.
   - Connect output to Active Routes Only and No Operation nodes.

6. **Create Filter Node (Active Routes Only)**
   - Condition: Only pass records where `Active` is true.
   - Input from Get Route by ID.
   - Output to Generate Actionable Task node.

7. **Create No Operation Node (No Operation, do nothing)**
   - Input from Get Route by ID (for inactive routes).
   - Output to Send notification about missing route.

8. **Create Generate Actionable Task Node (LangChain Agent)**
   - Input: Email text from Gmail Trigger.
   - Configure system prompt to extract one actionable task with title, description, optional bullet points.
   - Set output parser to Structured Output Parser1.
   - Connect OpenAI GPT-4o credentials.
   - Input from Active Routes Only.
   - Output to Get Summary & Meta Data.

9. **Create Structured Output Parser1 Node**
   - Parse JSON output from Generate Actionable Task.
   - Input from Generate Actionable Task.
   - Output to Get Summary & Meta Data.

10. **Create Get Summary & Meta Data Node (LangChain Agent)**
    - Input: Email text.
    - System prompt to generate detailed email summary and extract meta (sender, subject, date).
    - Set output parser to Structured Output Parser.
    - Connect OpenAI GPT-4o credentials.
    - Input from Generate Actionable Task.
    - Output to Format Notion Page Blocks.

11. **Create Structured Output Parser Node**
    - Parse JSON output from Get Summary & Meta Data.
    - Input from Get Summary & Meta Data.
    - Output to Format Notion Page Blocks.

12. **Create Format Notion Page Blocks Code Node**
    - JavaScript code to combine actionable task and summary/meta data.
    - Extract Notion Database ID from route config.
    - Format content into Notion page blocks (paragraphs, bullets, divider).
    - Output JSON body for Notion API.
    - Input from Get Summary & Meta Data.
    - Output to Create Notion Page.

13. **Create HTTP Request Node (Create Notion Page)**
    - Method: POST
    - URL: https://api.notion.com/v1/pages
    - Headers:
      - Authorization: Bearer token from route config.
      - Notion-Version: 2022-06-28
    - Body: JSON from Format Notion Page Blocks.
    - Retry on failure, wait 5 seconds between tries.
    - Input from Format Notion Page Blocks.
    - Output to Add Label "Processed" and Deactivate Route.

14. **Create Gmail Node (Add Label "Processed")**
    - Operation: Add Labels
    - Label ID: `processedLabelID` from Globals.
    - Message ID: from Gmail Trigger.
    - Input from Create Notion Page.

15. **Create Airtable Node (Deactivate Route)**
    - Operation: Update
    - Update field `Active` to false for the route ID.
    - Input from Create Notion Page error output.
    - Connect Airtable credentials.
    - Output to Send notification about deactivated route.

16. **Create Gmail Node (Send notification about deactivated route)**
    - Send email to original sender from email JSON.
    - Subject and body explaining route deactivation and troubleshooting.
    - Input from Deactivate Route.
    - Output to Add Label "Error".

17. **Create Gmail Node (Add Label "Error")**
    - Operation: Add Labels
    - Label ID: `errorLabelID`
    - Message ID: from Gmail Trigger.
    - Input from Send notification about deactivated route and Send notification about missing route.

18. **Create Gmail Node (Send notification about missing route)**
    - Send email to sender if no active route exists.
    - Input from No Operation node.
    - Output to Add Label "Error".

19. **Create Manual Trigger Node (When clicking ‘Test workflow’)**
    - Disabled by default.
    - Connect to Get all labels node.

20. **Create Gmail Node (Get all labels)**
    - Operation: List all Gmail labels.
    - Input from Manual Trigger.
    - Output to Required labels.

21. **Create Filter Node (Required labels)**
    - Filter labels containing “Processed” or “Error” in name.
    - Input from Get all labels.
    - Output to Globals.

22. **Populate Globals Node**
    - Copy label IDs from Required labels output into Globals node variables.
    - Use these IDs to configure Add Label nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Setup requires disabling Gmail Trigger and enabling manual trigger to fetch Gmail label IDs, then copying them to Globals. | See Sticky Note near Globals and manual trigger nodes in the workflow.                                                               |
| Airtable chosen for quick setup; self-hosted DB recommended for sensitive multi-user data.                                   | Disclaimer in workflow description.                                                                                                |
| Multi-user support enabled by alias-based routing and user-configurable routes exposed via Softr portal.                    | Workflow description and demo video: https://youtu.be/7cIvSqJAY0E                                                                   |
| Notion API requests are dynamically built due to dynamic auth tokens and variable content.                                  | Sticky Notes 6 and 7 explain this in detail.                                                                                         |
| AI processing splits task generation and summary/meta extraction into two agents for stability and clarity.                | Sticky Note 8 provides details.                                                                                                     |
| Gmail labels “Processed” and “Error” prevent double processing and track workflow status.                                   | Sticky Note 11; labels must be created manually in Gmail before setup.                                                              |
| Error handling includes route deactivation and user notification via email.                                                 | Sticky Note 10 covers notification strategy.                                                                                        |

---

This structured documentation enables advanced users and AI agents to understand, reproduce, and modify the workflow step-by-step, anticipate failure points, and manage integrations with Gmail, Airtable, OpenAI, and Notion APIs effectively.