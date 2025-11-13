Automatic Pantry Inventory Management with Notion and Email Notifications

https://n8nworkflows.xyz/workflows/automatic-pantry-inventory-management-with-notion-and-email-notifications-7918


# Automatic Pantry Inventory Management with Notion and Email Notifications

### 1. Workflow Overview

This workflow, titled **Automatic Pantry Inventory Management with Notion and Email Notifications**, automates the process of monitoring pantry stock levels stored in Notion and facilitates automatic replenishment by creating grocery list entries and sending email notifications. It targets users who maintain pantry inventories in Notion and want an automated reminder and list creation system to avoid running out of essential items.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Configuration:** Initiates the workflow on a daily schedule and sets user-specific configuration parameters.
- **1.2 Data Retrieval from Notion:** Fetches current pantry items and existing grocery list entries from Notion databases.
- **1.3 Computation of Refill Needs:** Processes the retrieved data to determine which items need restocking.
- **1.4 Grocery List Update & Notification:** Updates the grocery list in Notion with needed items and sends an email notification summarizing the refill requirements.
- **1.5 Error Handling (Disabled):** Contains an error trigger node, currently disabled, intended for managing runtime exceptions.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration

- **Overview:**  
  This block schedules the workflow to run daily and prepares any required user-specific configuration before proceeding.

- **Nodes Involved:**  
  - Daily Trigger  
  - User Config

- **Node Details:**

  - **Daily Trigger**  
    - Type: Cron Trigger  
    - Role: Automatically triggers the workflow once every day.  
    - Configuration: Uses default daily schedule (exact time not specified, relies on default cron settings).  
    - Inputs: None (trigger node).  
    - Outputs: Initiates "User Config" node.  
    - Edge cases: Cron misconfiguration or n8n downtime could delay workflow execution.

  - **User Config**  
    - Type: Set Node  
    - Role: Prepares and sets any static or dynamic configuration variables required downstream.  
    - Configuration: Empty parameters, likely a placeholder for future user variables or environment parameters.  
    - Inputs: Triggered by "Daily Trigger".  
    - Outputs: Passes data to both "Get Pantry Items" and "Get Existing To Buy" nodes.  
    - Edge cases: If configuration is incomplete or missing required parameters, downstream nodes may fail or behave unexpectedly.

#### 2.2 Data Retrieval from Notion

- **Overview:**  
  This block retrieves two sets of data from Notion: current pantry stock items and existing grocery list entries to avoid duplications.

- **Nodes Involved:**  
  - Get Pantry Items  
  - Get Existing To Buy

- **Node Details:**

  - **Get Pantry Items**  
    - Type: Notion Node (Database Read)  
    - Role: Retrieves pantry items from a Notion database to assess stock levels.  
    - Configuration: Connects to Notion with relevant credentials, queries pantry database (database ID and filters are set in node parameters but not explicitly shown).  
    - Inputs: Receives data from "User Config".  
    - Outputs: Sends pantry data to "Compute To-Buy List".  
    - Edge cases: API rate limits, authentication failures, or schema changes in Notion database could cause failures.

  - **Get Existing To Buy**  
    - Type: Notion Node (Database Read)  
    - Role: Retrieves existing grocery list items from Notion to check which items are already scheduled for purchase.  
    - Configuration: Similar to "Get Pantry Items", but targets grocery list database.  
    - Inputs: Receives data from "User Config".  
    - Outputs: Sends grocery list data to "Compute To-Buy List".  
    - Edge cases: Similar to "Get Pantry Items" — API limits, auth issues, or data inconsistency.

#### 2.3 Computation of Refill Needs

- **Overview:**  
  Processes the two datasets to identify which pantry items are below threshold and not already on the grocery list, thereby creating a refined list of items to be bought.

- **Nodes Involved:**  
  - Compute To-Buy List

- **Node Details:**

  - **Compute To-Buy List**  
    - Type: Function Node  
    - Role: Custom JavaScript logic that compares pantry stock against existing grocery list entries to generate a unique list of items to replenish.  
    - Configuration: Contains code to:  
      - Extract item quantities and thresholds from pantry data.  
      - Compare against grocery list items to avoid duplicates.  
      - Output an array of items requiring restocking.  
    - Inputs: Receives data from "Get Pantry Items" and "Get Existing To Buy".  
    - Outputs: Sends results to both "Create in Grocery List" and "Compose Email".  
    - Edge cases: Logic errors in script, empty datasets (which should be gracefully handled), potential undefined or malformed data structures.  
    - Version Requirements: Uses n8n Function Node v2 syntax.

#### 2.4 Grocery List Update & Notification

- **Overview:**  
  Updates the Notion grocery list database with new refill items and sends an email notification to the user summarizing these items.

- **Nodes Involved:**  
  - Create in Grocery List  
  - Compose Email  
  - Send Email

- **Node Details:**

  - **Create in Grocery List**  
    - Type: Notion Node (Database Create)  
    - Role: Adds the computed refill items as new entries into the grocery list database in Notion.  
    - Configuration: Uses Notion credentials, targets grocery list database, and maps item data fields accordingly.  
    - Inputs: Receives refill list from "Compute To-Buy List".  
    - Outputs: None downstream (terminal node for that branch).  
    - Edge cases: Failures due to rate limiting, data mapping errors, or permission issues.

  - **Compose Email**  
    - Type: Function Node  
    - Role: Builds a formatted email message body summarizing the items to be replenished.  
    - Configuration: Contains JavaScript code to format the list of items and quantities into a human-readable email content.  
    - Inputs: Receives refill item list from "Compute To-Buy List".  
    - Outputs: Passes composed email content to "Send Email".  
    - Edge cases: Formatting errors, empty item list should be handled to avoid sending empty emails.

  - **Send Email**  
    - Type: Email Send Node  
    - Role: Sends the composed grocery list email to the configured recipient.  
    - Configuration: Uses configured SMTP or email credentials (not specified, but required), email subject and body populated from "Compose Email".  
    - Inputs: Receives email content from "Compose Email".  
    - Outputs: None downstream (terminal node).  
    - Edge cases: SMTP authentication errors, connectivity issues, invalid email addresses.

#### 2.5 Error Handling (Disabled)

- **Overview:**  
  Intended to catch errors in the workflow execution and handle them gracefully.

- **Nodes Involved:**  
  - Error Handler

- **Node Details:**

  - **Error Handler**  
    - Type: Error Trigger Node  
    - Role: Captures any errors thrown during workflow execution to allow custom error handling or notifications.  
    - Configuration: Currently disabled, so no active role.  
    - Inputs: None (trigger node).  
    - Outputs: None specified.  
    - Edge cases: When enabled, must handle diverse error types including API failures, script errors, or network issues.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                       | Sticky Note                          |
|---------------------|---------------------|----------------------------------------|--------------------------------|------------------------------------|------------------------------------|
| Daily Trigger       | Cron Trigger        | Initiates workflow daily                | None                           | User Config                        |                                    |
| User Config         | Set                 | Sets user configuration variables      | Daily Trigger                  | Get Pantry Items, Get Existing To Buy |                                    |
| Get Pantry Items    | Notion (Database Read) | Retrieves pantry inventory data         | User Config                    | Compute To-Buy List                |                                    |
| Get Existing To Buy | Notion (Database Read) | Retrieves current grocery list entries  | User Config                    | Compute To-Buy List                |                                    |
| Compute To-Buy List | Function             | Determines items to restock              | Get Pantry Items, Get Existing To Buy | Create in Grocery List, Compose Email |                                    |
| Create in Grocery List | Notion (Database Create) | Adds refill items to grocery list        | Compute To-Buy List            | None                             |                                    |
| Compose Email       | Function             | Formats email content summarizing items | Compute To-Buy List            | Send Email                       |                                    |
| Send Email          | Email Send           | Sends notification email                 | Compose Email                  | None                             |                                    |
| Error Handler       | Error Trigger        | Handles workflow errors (disabled)       | None                          | None                             |                                    |
| ## Description      | Sticky Note          | Workflow description placeholder         | None                          | None                             |                                    |
| ## Setup Checklist  | Sticky Note          | Setup instructions placeholder           | None                          | None                             |                                    |
| ## Demo & Gallery   | Sticky Note          | Demo and gallery links placeholder       | None                          | None                             |                                    |
| ## QA / Test Plan   | Sticky Note          | Quality assurance and testing placeholder | None                          | None                             |                                    |
| ## Tags & Category  | Sticky Note          | Tags and categorization placeholder       | None                          | None                             |                                    |
| ## Attribution & License | Sticky Note      | Attribution and licensing placeholder      | None                          | None                             |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Pantry Soft-Stock Refill — Notion → Grocery List + Email".

2. **Add a Cron Trigger node** named "Daily Trigger":  
   - Set it to trigger once daily at your preferred time (default cron settings can be used).

3. **Add a Set node** named "User Config":  
   - Leave parameters empty initially or add any user variables needed for Notion database IDs or email addresses.

4. **Connect "Daily Trigger" output to "User Config" input**.

5. **Add a Notion node** named "Get Pantry Items":  
   - Set operation to "Retrieve Database Items".  
   - Provide Notion credentials with read access.  
   - Configure the node with the Pantry database ID.  
   - Apply any filters if needed to retrieve relevant pantry items.

6. **Add a Notion node** named "Get Existing To Buy":  
   - Set operation to "Retrieve Database Items".  
   - Use Notion credentials.  
   - Configure with the Grocery List database ID.  
   - Optionally filter to only active or pending items.

7. **Connect "User Config" output to both "Get Pantry Items" and "Get Existing To Buy" inputs**.

8. **Add a Function node** named "Compute To-Buy List":  
   - Paste JavaScript code that:  
     - Accepts input from both Notion queries.  
     - Compares pantry quantities against thresholds.  
     - Removes items already present in the grocery list.  
     - Outputs an array of items to replenish.  
   - Ensure the node uses version 2 syntax.

9. **Connect outputs of "Get Pantry Items" and "Get Existing To Buy" to "Compute To-Buy List" input** (multi-input supported).

10. **Add a Notion node** named "Create in Grocery List":  
    - Set operation to "Create Database Item".  
    - Use Notion credentials with write access.  
    - Configure to insert items into the grocery list database, mapping fields like item name and quantity from the function output.

11. **Add a Function node** named "Compose Email":  
    - Write JavaScript code to format the list of refill items into a readable email body (e.g., item names and quantities).

12. **Connect "Compute To-Buy List" output to both "Create in Grocery List" and "Compose Email" inputs**.

13. **Add an Email Send node** named "Send Email":  
    - Configure SMTP/email credentials (e.g., Gmail OAuth2 or SMTP server).  
    - Set recipient email address, subject line (e.g., "Pantry Refill List"), and body content using output from "Compose Email".

14. **Connect "Compose Email" output to "Send Email" input**.

15. **(Optional) Add an Error Trigger node** named "Error Handler":  
    - Enable and configure it to catch and handle workflow errors with notifications or logging.

16. **Test the workflow end-to-end** with sample data in the Notion databases to verify correct item retrieval, computation, list updating, and email notification.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                   |
|----------------------------------------------------------------------------------------------|---------------------------------|
| This workflow automates pantry stock monitoring and replenishment using Notion and email.    | Workflow Purpose                 |
| Requires valid Notion API credentials with read/write permissions on pantry and grocery list databases. | Credentials Setup                |
| Email node configuration requires SMTP or OAuth2 credentials for sending notifications.      | Email Setup                     |
| Consider enabling the error handler node to improve reliability and troubleshooting.         | Error Management Recommendation |
| For more on n8n Notion integration, visit: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion/ | Official n8n Notion Docs         |
| For best practices on email sending in n8n, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.emailSend/ | Official n8n Email Node Docs     |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.