Generate Weekly Grocery Lists in Notion with Automated Email Notifications

https://n8nworkflows.xyz/workflows/generate-weekly-grocery-lists-in-notion-with-automated-email-notifications-7427


# Generate Weekly Grocery Lists in Notion with Automated Email Notifications

### 1. Workflow Overview

This workflow automates the generation and distribution of a weekly grocery list based on predefined recipes, integrating with Notion for grocery list management and sending notifications via email and optionally Telegram. It is designed for busy homemakers, creators, and parents who want an automated, zero-cost meal planning assistant that runs on a weekly schedule.

The workflow consists of the following logical blocks:

- **1.1 Scheduling Trigger**: A Cron node schedules the workflow to run every Sunday at 6 PM.
- **1.2 Configuration Setup**: Defines core parameters such as recipe list, email addresses, Notion database IDs, and feature toggles.
- **1.3 Notion Database Validation**: Checks the accessibility of the configured Notion database before proceeding.
- **1.4 Grocery Item Generation**: Generates grocery list items from the recipes.
- **1.5 Notion Grocery List Update**: Adds the generated grocery items to the Notion grocery list database.
- **1.6 Notification Dispatch**: Sends the grocery list via email and optionally sends confirmation via Telegram.
- **1.7 Success Logging and Notifications**: Sends success email and optionally logs execution to Notion.
- **1.8 Error Handling and Notification**: Captures any workflow errors, formats error details, sends error notifications by email and optionally Telegram, and logs errors to Notion.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling Trigger

- **Overview:** Initiates the workflow automatically every Sunday at 6 PM to generate the weekly grocery list.
- **Nodes Involved:**  
  - `Cron: Weekly Meal Plan (Sun 6 PM)`

- **Node Details:**

  - **Cron: Weekly Meal Plan (Sun 6 PM)**
    - Type: Cron Trigger  
    - Configuration: Custom cron expression set to trigger at 18:00 every Sunday (`0 18 * * 0`)  
    - Input Connections: None (start node)  
    - Output Connections: Connects to `Set: Configuration`  
    - Edge Cases: If the cron schedule is changed incorrectly, the workflow may not run as intended. Testing mode is recommended (every minute) before setting weekly schedule.  

#### 2.2 Configuration Setup

- **Overview:** Defines all core variables including recipe list, email addresses, Notion database identifiers, and feature toggles to control optional notifications and logging.
- **Nodes Involved:**  
  - `Set: Configuration`

- **Node Details:**

  - **Set: Configuration**  
    - Type: Set  
    - Configuration: Sets string and boolean variables:
      - `recipeList`: A hardcoded list of 7 sample recipes formatted as emoji-labeled bullet points.
      - `emailTo`: Recipient email for grocery list.
      - `fromEmail`: Sender email address.
      - `notifyEmail`: Email for notifications and error alerts.
      - `notionDb`: Placeholder for Notion grocery list database ID.
      - `notionLogDb`: Placeholder for Notion logging database ID.
      - `telegramChatId`: Chat ID for Telegram notifications (optional).
      - `sendTelegram`: Boolean toggle for Telegram notifications.
      - `logToNotion`: Boolean toggle for persistent logging in Notion.
    - Input Connections: From `Cron: Weekly Meal Plan (Sun 6 PM)`
    - Output Connections: To `Notion: Validate Database Connection`
    - Edge Cases: Missing or incorrect Notion DB IDs or email addresses will cause failures downstream.

#### 2.3 Notion Database Validation

- **Overview:** Verifies connectivity and access to the configured Notion grocery list database before proceeding to generate grocery items.
- **Nodes Involved:**  
  - `Notion: Validate Database Connection`

- **Node Details:**

  - **Notion: Validate Database Connection**  
    - Type: Notion Database Query  
    - Configuration: Uses the Notion database ID from `notionDb` variable to query the database resource.  
    - Credentials: Requires valid Notion API credentials.  
    - Input Connections: From `Set: Configuration`  
    - Output Connections: To `Code: Generate Grocery Items`  
    - Edge Cases: Authentication failures, invalid database ID, or API rate limits can cause this node to fail, triggering error handling.

#### 2.4 Grocery Item Generation

- **Overview:** Processes the recipe list and generates individual grocery items to be added to Notion.
- **Nodes Involved:**  
  - `Code: Generate Grocery Items`

- **Node Details:**

  - **Code: Generate Grocery Items**  
    - Type: Code (JavaScript)  
    - Configuration: Contains custom JavaScript code that interprets the `recipeList` string and generates a structured list of grocery items (details of code not provided but implied).  
    - Input Connections: From `Notion: Validate Database Connection`  
    - Output Connections: To `Notion: Add to Grocery List`  
    - Edge Cases: Code errors or invalid recipe formatting can cause failures.

#### 2.5 Notion Grocery List Update

- **Overview:** Adds the generated grocery items as new pages in the configured Notion grocery list database.
- **Nodes Involved:**  
  - `Notion: Add to Grocery List`

- **Node Details:**

  - **Notion: Add to Grocery List**  
    - Type: Notion Database Page Creation  
    - Configuration: Creates new pages in the Notion database specified by `notionDb` with grocery item data from previous code node.  
    - Credentials: Uses Notion API credentials.  
    - Input Connections: From `Code: Generate Grocery Items`  
    - Output Connections: To `Email: Send Grocery List`  
    - Edge Cases: API failures, invalid data structure, or permission errors may cause failure and invoke error handling.

#### 2.6 Notification Dispatch

- **Overview:** Sends the grocery list via email and optionally sends a Telegram confirmation message.
- **Nodes Involved:**  
  - `Email: Send Grocery List`  
  - `IF: Telegram Enabled?`  
  - `Telegram: Confirmation`

- **Node Details:**

  - **Email: Send Grocery List**  
    - Type: Email Send  
    - Configuration: Sends an email with:
      - Subject: "🛒 This Week’s Grocery List"
      - To: `emailTo`
      - From: `fromEmail`
      - Body: Includes the recipe list formatted as plain text.
    - Credentials: Requires SMTP credentials configured externally.
    - Input Connections: From `Notion: Add to Grocery List`  
    - Output Connections: To `IF: Telegram Enabled?`, `Email: Send Success Notification`, and error handling.
    - Edge Cases: SMTP failures, incorrect email addresses, or network issues.

  - **IF: Telegram Enabled?**  
    - Type: If Condition  
    - Configuration: Checks if `sendTelegram` boolean is true to decide whether to send Telegram confirmation.  
    - Input Connections: From `Email: Send Grocery List`  
    - Output Connections: To `Telegram: Confirmation` (true branch) and error handling (false branch).

  - **Telegram: Confirmation**  
    - Type: Telegram Message  
    - Configuration: Sends a confirmation message "🛒 Grocery list emailed! Happy cooking 🌸" to the chat ID specified by `telegramChatId`.  
    - Credentials: Requires Telegram bot credentials.  
    - Input Connections: From `IF: Telegram Enabled?`  
    - Output Connections: None  
    - Edge Cases: Telegram API failures, invalid chat ID, or disabling Telegram notifications.

#### 2.7 Success Logging and Notifications

- **Overview:** Sends a success notification email and optionally logs the successful execution to a Notion database.
- **Nodes Involved:**  
  - `Email: Send Success Notification`  
  - `IF: Log to Notion?`  
  - `Notion: Append Log Entry (Success)`  
  - `Set: Log Workflow Execution`

- **Node Details:**

  - **Email: Send Success Notification**  
    - Type: Email Send  
    - Configuration: Sends a success email with timestamp, recipe list, and grocery list summary to `notifyEmail`.  
    - Input Connections: From `Email: Send Grocery List`  
    - Output Connections: To `IF: Log to Notion?`  
    - Edge Cases: SMTP or email delivery issues.

  - **IF: Log to Notion?**  
    - Type: If Condition  
    - Configuration: Checks if `logToNotion` boolean is true to decide whether to log the execution.  
    - Input Connections: From `Email: Send Success Notification`  
    - Output Connections: To `Notion: Append Log Entry (Success)` (true branch)

  - **Notion: Append Log Entry (Success)**  
    - Type: Notion Database Page Creation  
    - Configuration: Adds a log entry with status "success" and current timestamp to the Notion log database (`notionLogDb`).  
    - Input Connections: From `IF: Log to Notion?`  
    - Edge Cases: API permission or connectivity issues.

  - **Set: Log Workflow Execution**  
    - Type: Set  
    - Configuration: Sets basic status and timestamp values (used for logging).  
    - Input Connections: None explicitly shown but likely used in logging context.

#### 2.8 Error Handling and Notification

- **Overview:** Captures errors from upstream nodes, normalizes error details, notifies stakeholders via email and optionally Telegram, and logs errors in Notion.
- **Nodes Involved:**  
  - `Catch: Error Handling`  
  - `Format: Error Payload`  
  - `Email: Send Error Notification`  
  - `IF: Telegram Enabled (Error)?`  
  - `Telegram: Error Notification`  
  - `Notion: Append Log Entry (Error)`

- **Node Details:**

  - **Catch: Error Handling**  
    - Type: Set (Error Catcher)  
    - Configuration: Receives error data from any upstream node failure.  
    - Input Connections: Error output from nodes like `Notion: Add to Grocery List`, `Email: Send Grocery List`, etc.  
    - Output Connections: To `Format: Error Payload`

  - **Format: Error Payload**  
    - Type: Code (JavaScript)  
    - Configuration: Normalizes error message, node name, timestamp, and optionally stack trace for consistent error reporting.  
    - Input Connections: From `Catch: Error Handling`  
    - Output Connections: To `Email: Send Error Notification`, `IF: Telegram Enabled (Error)?`, and `Notion: Append Log Entry (Error)`

  - **Email: Send Error Notification**  
    - Type: Email Send  
    - Configuration: Sends an error notification email with detailed error information to `notifyEmail`.  
    - Input Connections: From `Format: Error Payload`  
    - Credentials: Requires SMTP  
    - Output Connections: None  

  - **IF: Telegram Enabled (Error)?**  
    - Type: If Condition  
    - Configuration: Checks `sendTelegram` to decide if Telegram error notification is sent.  
    - Input Connections: From `Format: Error Payload`  
    - Output Connections: To `Telegram: Error Notification` (true branch)

  - **Telegram: Error Notification**  
    - Type: Telegram Message  
    - Configuration: Sends an error alert with node name, message, and timestamp to `telegramChatId`.  
    - Input Connections: From `IF: Telegram Enabled (Error)?`  
    - Credentials: Requires Telegram credentials.  
    - Output Connections: None  

  - **Notion: Append Log Entry (Error)**  
    - Type: Notion Database Page Creation  
    - Configuration: Adds an error log entry to the Notion log database (`notionLogDb`) with error details.  
    - Input Connections: From `Format: Error Payload`  
    - Edge Cases: API permission issues or insufficient logging database access.

---

### 3. Summary Table

| Node Name                          | Node Type                 | Functional Role                          | Input Node(s)                     | Output Node(s)                                   | Sticky Note                                                                                                      |
|-----------------------------------|---------------------------|----------------------------------------|----------------------------------|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| README – Template Description     | Sticky Note               | Documentation and usage overview       | None                             | None                                            | Describes the template purpose, setup, customization, and reliability notes                                      |
| Cron: Weekly Meal Plan (Sun 6 PM) | Cron                      | Workflow trigger scheduled weekly      | None                             | Set: Configuration                              |                                                                                                                  |
| Set: Configuration                | Set                       | Defines configuration variables        | Cron: Weekly Meal Plan            | Notion: Validate Database Connection            |                                                                                                                  |
| Notion: Validate Database Connection | Notion                   | Validates Notion DB access             | Set: Configuration                | Code: Generate Grocery Items                     | Checks access to the Notion database ID before creating pages                                                    |
| Code: Generate Grocery Items      | Code                      | Generates grocery items from recipes   | Notion: Validate Database Connection | Notion: Add to Grocery List                     |                                                                                                                  |
| Notion: Add to Grocery List       | Notion                    | Adds grocery items to Notion DB        | Code: Generate Grocery Items      | Email: Send Grocery List                         |                                                                                                                  |
| Email: Send Grocery List          | Email Send                | Sends grocery list email                | Notion: Add to Grocery List       | IF: Telegram Enabled?, Email: Send Success Notification, Catch: Error Handling |                                                                                                                  |
| IF: Telegram Enabled?             | If                        | Checks if Telegram notifications enabled | Email: Send Grocery List          | Telegram: Confirmation, Catch: Error Handling    |                                                                                                                  |
| Telegram: Confirmation            | Telegram                  | Sends Telegram confirmation message    | IF: Telegram Enabled?             | None                                            |                                                                                                                  |
| Email: Send Success Notification | Email Send                | Sends success notification email       | Email: Send Grocery List          | IF: Log to Notion?                              | Optional success notice; can be disabled before submit if you prefer                                             |
| IF: Log to Notion?                | If                        | Checks if logging to Notion is enabled | Email: Send Success Notification | Notion: Append Log Entry (Success)               |                                                                                                                  |
| Notion: Append Log Entry (Success) | Notion                   | Logs successful execution in Notion    | IF: Log to Notion?                | None                                            | Optional persistent logging to Notion                                                                            |
| Set: Log Workflow Execution       | Set                       | Sets log status and timestamp          | None                             | None                                            | Placeholder logger. Replace with Notion/Sheets if persistent logs desired                                         |
| Catch: Error Handling             | Set                       | Captures errors for notification       | Multiple upstream error outputs  | Format: Error Payload                            | Receives error output from upstream nodes and forwards to error email                                            |
| Format: Error Payload             | Code                      | Normalizes error details for reporting | Catch: Error Handling             | Email: Send Error Notification, IF: Telegram Enabled (Error) , Notion: Append Log Entry (Error) | Normalizes error details for notifications and logging                                                            |
| Email: Send Error Notification   | Email Send                | Sends error notification email         | Format: Error Payload             | None                                            | Attach SMTP credentials after import                                                                             |
| IF: Telegram Enabled (Error)?    | If                        | Checks if Telegram error notifications enabled | Format: Error Payload             | Telegram: Error Notification                      |                                                                                                                  |
| Telegram: Error Notification      | Telegram                  | Sends Telegram error alert              | IF: Telegram Enabled (Error)?    | None                                            | Optional Telegram error alert                                                                                     |
| Notion: Append Log Entry (Error)  | Notion                    | Logs error in Notion                    | Format: Error Payload             | None                                            |                                                                                                                  |
| Setup Tips                       | Sticky Note               | Quick setup and tuning instructions    | None                             | None                                            | Quick Tune-ups and testing instructions                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Cron Trigger node:**
   - Name: `Cron: Weekly Meal Plan (Sun 6 PM)`
   - Set mode to "Custom"
   - Enter cron expression: `0 18 * * 0` (Every Sunday at 18:00)

3. **Add a Set node:**
   - Name: `Set: Configuration`
   - Add the following string fields:
     - `recipeList` with the 7 sample recipes as a multiline string:
       ```
       - 🌸 Creamy Tuscan Chicken
       - 🥗 Lemon-Garlic Shrimp Salad
       - 🍝 One-Pan Veggie Pasta
       - 🥦 Teriyaki Salmon & Rice
       - 🍕 Margherita Flatbread
       - 🍫 Dark-Choc Overnight Oats
       - 🥒 Cucumber-Lime Smoothie
       ```
     - `emailTo`: recipient email (e.g., `recipient@example.com`)
     - `fromEmail`: sender email (e.g., `sender@example.com`)
     - `notifyEmail`: email for notifications (e.g., `owner@example.com`)
     - `notionDb`: placeholder for your Notion grocery list database ID
     - `notionLogDb`: placeholder for your Notion log database ID
     - `telegramChatId`: (optional) your Telegram chat ID if using Telegram notifications
   - Add boolean fields:
     - `sendTelegram`: (default false) to enable Telegram notifications
     - `logToNotion`: (default false) to enable execution logging in Notion

4. **Connect `Cron: Weekly Meal Plan` node to `Set: Configuration`.**

5. **Add a Notion node:**
   - Name: `Notion: Validate Database Connection`
   - Resource: Database
   - Database ID: expression `{{$json.notionDb}}`
   - Use your Notion API credentials.
   - Connect `Set: Configuration` → `Notion: Validate Database Connection`

6. **Add a Code node:**
   - Name: `Code: Generate Grocery Items`
   - Write JavaScript to parse the `recipeList` string and generate grocery items.
   - Input: use `$json.recipeList`
   - Output: structured data representing grocery items.
   - Connect `Notion: Validate Database Connection` → `Code: Generate Grocery Items`

7. **Add a Notion node:**
   - Name: `Notion: Add to Grocery List`
   - Resource: Database Page
   - Database ID: expression `{{$json.notionDb}}`
   - Use Notion API credentials.
   - Configure to create pages for each grocery item generated.
   - Connect `Code: Generate Grocery Items` → `Notion: Add to Grocery List`

8. **Add an Email Send node:**
   - Name: `Email: Send Grocery List`
   - Configure SMTP credentials.
   - To Email: `{{$json.emailTo}}`
   - From Email: `{{$json.fromEmail}}`
   - Subject: "🛒 This Week’s Grocery List"
   - Body: Include `{{$json.recipeList}}` in the email text.
   - Connect `Notion: Add to Grocery List` → `Email: Send Grocery List`

9. **Add an If node:**
   - Name: `IF: Telegram Enabled?`
   - Condition: Boolean equals true for `$json.sendTelegram`
   - Connect `Email: Send Grocery List` → `IF: Telegram Enabled?`

10. **Add a Telegram node:**
    - Name: `Telegram: Confirmation`
    - Text: "🛒 Grocery list emailed! Happy cooking 🌸"
    - Chat ID: `{{$json.telegramChatId}}`
    - Use Telegram credentials.
    - Connect `IF: Telegram Enabled?` (true) → `Telegram: Confirmation`

11. **Add an Email Send node:**
    - Name: `Email: Send Success Notification`
    - To: `{{$json.notifyEmail}}`
    - From: `{{$json.fromEmail}}`
    - Subject: "✅ Meal Planner – Success"
    - Body: Include timestamp, recipe list, and grocery list summary.
    - Connect `Email: Send Grocery List` → `Email: Send Success Notification`

12. **Add an If node:**
    - Name: `IF: Log to Notion?`
    - Condition: Boolean equals true for `$json.logToNotion`
    - Connect `Email: Send Success Notification` → `IF: Log to Notion?`

13. **Add a Notion node:**
    - Name: `Notion: Append Log Entry (Success)`
    - Resource: Database Page
    - Database ID: expression `{{$json.notionLogDb}}`
    - Creates a log entry with status "success" and current timestamp.
    - Connect `IF: Log to Notion?` (true) → `Notion: Append Log Entry (Success)`

14. **Add a Set node:**
    - Name: `Catch: Error Handling`
    - Configure this node to be an error catcher for upstream nodes.
    - Connect error outputs from critical nodes (Notion, Email, Telegram) to this node.

15. **Add a Code node:**
    - Name: `Format: Error Payload`
    - Normalize error details: node name, message, timestamp, stack trace.
    - Connect `Catch: Error Handling` → `Format: Error Payload`

16. **Add an Email Send node:**
    - Name: `Email: Send Error Notification`
    - To: `{{$json.notifyEmail}}`
    - From: `{{$json.fromEmail}}`
    - Subject: "❗ Meal Planner – Error in {{$json.errorNode}}"
    - Body: Include error details.
    - Connect `Format: Error Payload` → `Email: Send Error Notification`

17. **Add an If node:**
    - Name: `IF: Telegram Enabled (Error)?`
    - Condition: Boolean equals true for `$json.sendTelegram`
    - Connect `Format: Error Payload` → `IF: Telegram Enabled (Error)?`

18. **Add a Telegram node:**
    - Name: `Telegram: Error Notification`
    - Text: "⚠️ Meal Planner error in {{$json.errorNode}}:\n{{$json.errorMessage}}\n{{$json.timestamp}}"
    - Chat ID: `{{$json.telegramChatId}}`
    - Use Telegram credentials.
    - Connect `IF: Telegram Enabled (Error)?` (true) → `Telegram: Error Notification`

19. **Add a Notion node:**
    - Name: `Notion: Append Log Entry (Error)`
    - Resource: Database Page
    - Database ID: expression `{{$json.notionLogDb}}`
    - Creates an error log entry with error details.
    - Connect `Format: Error Payload` → `Notion: Append Log Entry (Error)`

20. **Add Sticky Notes:**
    - Add descriptive sticky notes for setup tips and template overview as per original.

21. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                             | Context or Link                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Runs every Sunday at 6 PM by default but can be customized by editing the Cron node. For testing, set the Cron to run every minute, then revert before activating.                                                                                                                        | See "README – Template Description" and "Setup Tips" sticky notes.                                                 |
| Requires free Notion account and setup of Notion API credentials linked to your Meal Planner database and optionally a logging database.                                                                                                                                                 | Notion API documentation: https://developers.notion.com/                                                            |
| SMTP email or Telegram credentials must be configured externally in n8n for sending notifications.                                                                                                                                                                                     | n8n Credential setup for SMTP and Telegram: https://docs.n8n.io/credentials/                                          |
| The workflow includes error handling that sends immediate notifications on failure with detailed error information and attempts persistent logging if enabled.                                                                                                                           |                                                                                                                    |
| The recipe list can be customized by editing the `recipeList` variable in the `Set: Configuration` node. Additional customization includes adding dietary tags or portion sizes in Notion database schemas.                                                                                |                                                                                                                    |
| Optional: To maintain persistent logs, set `logToNotion` to true and provide a valid Notion database ID for `notionLogDb`.                                                                                                                                                               |                                                                                                                    |
| Telegram notification is optional and controlled by toggling `sendTelegram` and providing a valid `telegramChatId`.                                                                                                                                                                      |                                                                                                                    |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.