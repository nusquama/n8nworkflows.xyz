Automate Stripe Invoicing Workflow with ClickUp & Gmail Notifications

https://n8nworkflows.xyz/workflows/automate-stripe-invoicing-workflow-with-clickup---gmail-notifications-9542


# Automate Stripe Invoicing Workflow with ClickUp & Gmail Notifications

### 1. Workflow Overview

This workflow automates the invoicing process by integrating ClickUp task status updates, Stripe invoicing, and Gmail notifications. It listens for changes in ClickUp task statuses, and when a task status changes to "send invoice," it triggers a series of operations to create a customer in Stripe, generate an invoice and invoice item, send the invoice to the customer, and notify the internal team via email.

**Logical blocks:**

- **1.1 Input Reception:** Monitor ClickUp task status updates via webhook trigger.
- **1.2 Status Verification:** Check if the new status is "send invoice" to proceed.
- **1.3 Task Data Retrieval:** Fetch detailed task information from ClickUp.
- **1.4 Stripe Customer Creation:** Create a new customer in Stripe based on task creator data.
- **1.5 Invoice and Invoice Item Creation:** Generate an invoice and invoice item in Stripe.
- **1.6 Invoice Dispatch:** Send the created invoice to the customer.
- **1.7 Team Notification:** Send an email notification to the team that the invoice was sent.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception — ClickUp Status Change Trigger

- **Overview:** Listens for updates on task statuses in a specific ClickUp list.
- **Nodes Involved:** ClickUp Trigger
- **Node Details:**

  - **ClickUp Trigger**
    - Type: ClickUp Trigger node; listens for task events.
    - Configuration: Authenticated via OAuth2; triggers on "taskStatusUpdated" events filtered to a specific list ID.
    - Key Expression: None (event-driven).
    - Input Connections: None (trigger node).
    - Output Connections: Connects to "Status = send invoice".
    - Version: 1.
    - Edge Cases: OAuth2 token expiration or permission errors; webhook delivery delays; changes outside specified list ignored.

#### 2.2 Status Verification — Conditional Branching on Status

- **Overview:** Validates if the task status update corresponds to "send invoice" to continue workflow.
- **Nodes Involved:** Status = send invoice (If node)
- **Node Details:**

  - **Status = send invoice**
    - Type: If node; conditional logic.
    - Configuration: Checks if `history_items[0].after.status` equals "send invoice".
    - Key Expression: `={{ $json.history_items[0].after.status }} == "send invoice"`.
    - Input Connections: From ClickUp Trigger.
    - Output Connections: True branch leads to "ClickUp" node; no false branch continuation.
    - Version: 2.2.
    - Edge Cases: Missing or malformed status data in payload; status case sensitivity; multiple status updates in history array.

#### 2.3 Task Data Retrieval — Fetch Full Task Data

- **Overview:** Retrieves the full details of the ClickUp task that triggered the event.
- **Nodes Involved:** ClickUp
- **Node Details:**

  - **ClickUp**
    - Type: ClickUp node; performs API operation.
    - Configuration: Authenticated via OAuth2; operation "get" on task ID from trigger.
    - Key Expression: `={{ $('ClickUp Trigger').item.json.task_id }}` for task ID.
    - Input Connections: From "Status = send invoice" (true path).
    - Output Connections: Connects to "Create Customer".
    - Version: 1.
    - Edge Cases: Task deleted or inaccessible; OAuth2 token expired; API rate limits.

#### 2.4 Stripe Customer Creation

- **Overview:** Creates a new customer in Stripe using task creator's name and email.
- **Nodes Involved:** Create Customer
- **Node Details:**

  - **Create Customer**
    - Type: Stripe node; resource "customer", operation "create".
    - Configuration: Customer name from task name, email from `creator.email` custom field of the ClickUp task.
    - Key Expressions: 
      - Name: `={{ $json.name }}`
      - Email: `={{ $json.creator.email }}`
    - Input Connections: From "ClickUp".
    - Output Connections: Connects to "Create Invoice".
    - Version: 1.
    - Credential: Stripe API credentials required.
    - Edge Cases: Email missing or invalid; Stripe API errors; duplicate customers.

#### 2.5 Invoice and Invoice Item Creation

- **Overview:** Generates an invoice and invoice item in Stripe for the customer created.
- **Nodes Involved:** Create Invoice, Create item invoice
- **Node Details:**

  - **Create Invoice**
    - Type: HTTP Request node calling Stripe API.
    - Configuration:
      - POST to `https://api.stripe.com/v1/invoices`
      - Query parameters:
        - `collection_method`: "send_invoice"
        - `customer`: customer ID from previous node (`={{ $json.id }}`)
        - `description`: fixed string "Thanks for working with SaviFlow!"
        - `due_date`: current timestamp + 7 days (`={{ $today.toSeconds() + 604800 }}`)
        - `footer`: fixed string "This is the very cool footer"
      - Authentication: Stripe API credentials.
    - Input Connections: From "Create Customer".
    - Output Connections: Connects to "Create item invoice".
    - Version: 4.2.
    - Edge Cases: Customer ID missing; API rate limits; date calculation errors.

  - **Create item invoice**
    - Type: HTTP Request node calling Stripe API.
    - Configuration:
      - POST to `https://api.stripe.com/v1/invoiceitems`
      - Query parameters:
        - `customer`: customer ID from "Create Customer" (`={{ $('Create Customer').last().json.id }}`)
        - `amount`: amount from ClickUp custom field index 1 multiplied by 100 (to convert to cents).
        - `description`: fixed string "Thanks for building the coolest automation with SaviFlow"
        - `currency`: from JSON `currency` field.
        - `invoice`: invoice ID from current JSON (`={{ $json.id }}`).
      - Authentication: Stripe API credentials.
    - Input Connections: From "Create Invoice".
    - Output Connections: Connects to "Send invoice".
    - Version: 4.2.
    - Edge Cases: Missing or invalid amounts; currency not set; invoice ID missing; API errors.

#### 2.6 Invoice Dispatch

- **Overview:** Sends the created invoice to the customer via Stripe.
- **Nodes Involved:** Send invoice
- **Node Details:**

  - **Send invoice**
    - Type: HTTP Request node calling Stripe API.
    - Configuration:
      - POST to `https://api.stripe.com/v1/invoices/{{invoice_id}}/send` where `invoice_id` is from "Create Invoice" node.
      - Authentication: Stripe API credentials.
    - Input Connections: From "Create item invoice".
    - Output Connections: Connects to "Send email to you/team".
    - Version: 4.2.
    - Edge Cases: Invoice ID missing; API errors; invoice already sent.

#### 2.7 Team Notification — Send Email via Gmail

- **Overview:** Sends an email notification to a team member or user about the new invoice.
- **Nodes Involved:** Send email to you/team
- **Node Details:**

  - **Send email to you/team**
    - Type: Gmail node; sends email.
    - Configuration:
      - Recipient: Email address from ClickUp task custom field index 0.
      - Subject: "Yay! New invoice has been sent".
      - Message Body: Personalized message including customer name and ClickUp task link.
      - Email type: Text only; no attribution appended.
    - Input Connections: From "Send invoice".
    - Output Connections: None (terminal node).
    - Version: 2.1.
    - Credential: Gmail OAuth2 credentials.
    - Edge Cases: Invalid email; Gmail API limits; OAuth token expiration.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                        | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                           |
|------------------------|---------------------|-------------------------------------|------------------------|--------------------------|-----------------------------------------------------------------------------------------------------|
| ClickUp Trigger        | ClickUp Trigger      | Listen for ClickUp task status updates | None                   | Status = send invoice     | Watches ClickUp For Updated Status. Recommend testing Stripe in "Test Mode" before live usage.       |
| Status = send invoice  | If                  | Check if status equals "send invoice" | ClickUp Trigger        | ClickUp                  |                                                                                                     |
| ClickUp                | ClickUp              | Fetch full task details               | Status = send invoice   | Create Customer          | Gets the task that was updated and gets all of that task's info for invoicing.                       |
| Create Customer        | Stripe              | Create Stripe customer                 | ClickUp                 | Create Invoice           | Creates a new customer in your Stripe Account                                                      |
| Create Invoice         | HTTP Request        | Create an invoice in Stripe            | Create Customer         | Create item invoice      | Invoice creation and sending the invoice                                                           |
| Create item invoice    | HTTP Request        | Create an invoice item in Stripe       | Create Invoice          | Send invoice             | Invoice creation and sending the invoice                                                           |
| Send invoice           | HTTP Request        | Send the Stripe invoice to customer    | Create item invoice     | Send email to you/team   | Invoice creation and sending the invoice                                                           |
| Send email to you/team | Gmail               | Notify team about invoice sent          | Send invoice            | None                     | Notify's you or your team that a new invoice has been sent                                         |
| Sticky Note            | Sticky Note         | Instructional notes                     | None                   | None                     | Watches ClickUp For Updated Status. Recommend Stripe in "Test Mode" for initial runs.               |
| Sticky Note1           | Sticky Note         | Instructional note                      | None                   | None                     | Invoice creation and Sending the invoice                                                           |
| Sticky Note2           | Sticky Note         | Instructional note                      | None                   | None                     | Notify's you or your team that a new invoice has been sent                                         |
| Sticky Note3           | Sticky Note         | Instructional note                      | None                   | None                     | Creates a new customer in your Stripe Account                                                      |
| Sticky Note4           | Sticky Note         | Instructional note                      | None                   | None                     | Gets the task that was updated and gets all of that tasks info to be processed in the invoice       |
| Sticky Note5           | Sticky Note         | Social media links                      | None                   | None                     | Find my Socials: YouTube and LinkedIn links                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the ClickUp Trigger node:**
   - Type: ClickUp Trigger
   - Configure OAuth2 credentials for ClickUp.
   - Set Team ID to `90151078626`.
   - Set event to trigger on `taskStatusUpdated`.
   - Add filter for List ID `901510285394`.
   - Position it as the starting node.

2. **Add an If node for status verification:**
   - Name: Status = send invoice
   - Condition: Check if `history_items[0].after.status` equals `"send invoice"`.
   - Connect ClickUp Trigger main output to this node.

3. **Create ClickUp node to get full task data:**
   - Type: ClickUp
   - Configure OAuth2 credentials.
   - Set operation to "get".
   - Use expression for Task ID: `={{ $json.task_id }}` from trigger, or specifically `={{ $('ClickUp Trigger').item.json.task_id }}`.
   - Connect true output of If node to this node.

4. **Add Stripe node to create customer:**
   - Type: Stripe
   - Resource: Customer
   - Operation: Create
   - Set Name from task name: `={{ $json.name }}`
   - Set Email from `creator.email` field: `={{ $json.creator.email }}`
   - Connect output of ClickUp node to this.

5. **Create HTTP Request node to create Stripe invoice:**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://api.stripe.com/v1/invoices`
   - Authentication: Stripe API credentials
   - Query Parameters:
     - `collection_method`: "send_invoice"
     - `customer`: `={{ $json.id }}` (customer ID from previous node)
     - `description`: "Thanks for working with SaviFlow!"
     - `due_date`: `={{ $today.toSeconds() + 604800 }}` (7 days from now)
     - `footer`: "This is the very cool footer"
   - Connect output of Stripe customer creation to this node.

6. **Create HTTP Request node for invoice item creation:**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://api.stripe.com/v1/invoiceitems`
   - Authentication: Stripe API credentials
   - Query Parameters:
     - `customer`: `={{ $('Create Customer').last().json.id }}`
     - `amount`: `={{ $('ClickUp').last().json.custom_fields[1].value * 100 }}`
     - `description`: "Thanks for building the coolest automation with SaviFlow"
     - `currency`: `={{ $json.currency }}`
     - `invoice`: `={{ $json.id }}`
   - Connect output of "Create Invoice" node to this.

7. **Create HTTP Request node to send the invoice:**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://api.stripe.com/v1/invoices/{{invoice_id}}/send`
     - Use expression: `=https://api.stripe.com/v1/invoices/{{ $('Create Invoice').item.json.id }}/send`
   - Authentication: Stripe API credentials
   - Connect output of "Create item invoice" node to this.

8. **Create Gmail node to send notification email:**
   - Type: Gmail
   - Configure Gmail OAuth2 credentials.
   - Send To: `={{ $('ClickUp').item.json.custom_fields[0].value }}`
   - Subject: "Yay! New invoice has been sent"
   - Message: 
     ```
     Hi Seb,
     
     Congrats! You have sent a new invoice to {{ $('ClickUp').item.json.name }}
     
     View the task: https://app.clickup.com/t/{{ $('ClickUp Trigger').item.json.task_id }}
     ```
   - Email Type: Text
   - Do not append attribution.
   - Connect output of "Send invoice" node to this.

9. **Add Sticky Notes for documentation:**
   - Add notes for ClickUp Trigger explaining test mode recommendation.
   - Add notes for invoice creation steps.
   - Add notes for notification step.
   - Add notes for customer creation.
   - Add notes for task data retrieval.
   - Add social media links note.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Recommend putting your Stripe account in "Test Mode" when running the workflow for the first time to avoid risks. | Sticky note near ClickUp Trigger node.                                                         |
| Find my Socials: YouTube: https://www.youtube.com/@SebGardners, LinkedIn: https://www.linkedin.com/in/seb-gardner-5b439a260/ | Sticky note with social media links for workflow author.                                       |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automated workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.