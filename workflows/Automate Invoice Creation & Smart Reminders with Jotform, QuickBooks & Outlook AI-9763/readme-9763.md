Automate Invoice Creation & Smart Reminders with Jotform, QuickBooks & Outlook AI

https://n8nworkflows.xyz/workflows/automate-invoice-creation---smart-reminders-with-jotform--quickbooks---outlook-ai-9763


# Automate Invoice Creation & Smart Reminders with Jotform, QuickBooks & Outlook AI

### 1. Workflow Overview

This workflow automates the process of generating invoices and sending payment reminders by integrating Jotform, QuickBooks Online (QBO), and Outlook email services. It is designed to:

- Receive customer order and billing information via Jotform submissions.
- Verify if the customer exists in QuickBooks; update or create customer records accordingly.
- Retrieve product/service details and create corresponding invoices in QuickBooks.
- Email the generated invoices automatically to customers.
- Store invoice metadata in a database to track payment status and reminders.
- Schedule and send invoice payment reminders based on configurable intervals.
- Summarize daily reminder activities using an AI agent and send a report email to the internal team.

The workflow is logically divided into the following blocks:

- **1.1 Receive and Format Input Data**: Triggered by a Jotform webhook, formats submission data.
- **1.2 Customer Verification and Management**: Checks if the customer exists in QuickBooks, updates or creates customer data.
- **1.3 Product Retrieval and Invoice Generation**: Fetches product info, creates a QuickBooks invoice, and sends it by email.
- **1.4 Invoice Data Persistence**: Inserts invoice info into a database table for tracking.
- **1.5 Scheduled Reminder Trigger and Configuration**: A daily trigger initiates the reminder process with configurable intervals.
- **1.6 Invoice Reminder Logic and Dispatch**: Processes invoices to decide on sending reminders, skipping, or deleting records.
- **1.7 AI-Based Summary and Email Report**: Uses AI to summarize sent reminders and emails a daily report.

---

### 2. Block-by-Block Analysis

#### 1.1 Receive and Format Input Data

**Overview:**  
Receives incoming form submissions from Jotform via webhook and formats the raw data for downstream processing.

**Nodes Involved:**  
- Receive form submission  
- Format data  
- Sticky Note (describing this block)

**Node Details:**

- **Receive form submission**  
  - Type: Webhook  
  - Role: Entry point triggered by HTTP POST from Jotform webhook.  
  - Config: Path set to a unique webhook ID; method POST.  
  - Inputs: External HTTP POST request with form data.  
  - Outputs: Raw JSON containing customer and order details.  
  - Potential Failures: Webhook misconfiguration, invalid payloads.

- **Format data**  
  - Type: Code (JavaScript)  
  - Role: Parses billing address from HTML string, extracts customer name, email, phone, and item name.  
  - Config: Custom JS regex-based parser for billing address fields.  
  - Expressions: Uses `$input.first().json.body` to extract fields.  
  - Inputs: Raw webhook JSON.  
  - Outputs: Structured JSON with `address`, `customer`, and `item` objects.  
  - Potential Failures: Regex parsing errors if billing address format changes or missing fields.

- **Sticky Note**: Describes this block as "Receive Submission" with notes about Jotform integration.

---

#### 1.2 Customer Verification and Management

**Overview:**  
Checks if the customer exists in QuickBooks by email; updates billing details if found, or creates a new customer record if not.

**Nodes Involved:**  
- Check if the customer exists  
- If (conditional branching)  
- Update the customer  
- Create the customer  
- Add customer id (code)  
- Sticky Notes (Check customer exists, Customer exists, Customer doesn’t exist, Add customer id)

**Node Details:**

- **Check if the customer exists**  
  - Type: QuickBooks (getAll)  
  - Role: Queries QuickBooks for customers filtered by primary email address.  
  - Config: Filter query `Where PrimaryEmailAddr = '{{ $json.customer.email }}'`  
  - Inputs: Formatted data JSON with customer email.  
  - Outputs: Customer record if exists, empty otherwise.  
  - Edge cases: API rate limits, auth errors, missing email field.

- **If (Condition)**  
  - Type: If  
  - Role: Branches workflow based on presence of customer record and valid customer ID.  
  - Condition: Checks if `$json` exists and `$json.Id` exists.  
  - Outputs:  
    - True: Customer exists → Update customer  
    - False: Customer doesn't exist → Create customer

- **Update the customer**  
  - Type: QuickBooks (update)  
  - Role: Updates customer billing address, name, and phone in QuickBooks.  
  - Config: Uses customer ID from previous node, updates billing address fields from formatted data.  
  - Inputs: Customer record from check node.  
  - Outputs: Updated customer record.  
  - Failures: Invalid customer ID, API errors.

- **Create the customer**  
  - Type: QuickBooks (create)  
  - Role: Creates a new customer in QuickBooks with billing details and contact info.  
  - Config: Uses formatted customer and address data.  
  - Outputs: Newly created customer record with ID.  
  - Failures: Validation errors on required fields, API limits.

- **Add customer id**  
  - Type: Code (JavaScript)  
  - Role: Adds the QuickBooks customer ID back into the JSON data for downstream use.  
  - Inputs: Output from either update or create customer node.  
  - Outputs: JSON augmented with `customer.id`.  
  - Edge cases: Missing or invalid ID.

- **Sticky Notes**: Explain the logic of checking and handling customer existence.

---

#### 1.3 Product Retrieval and Invoice Generation

**Overview:**  
Fetches the product/service item from QuickBooks, creates an invoice for the customer, and sends it via email.

**Nodes Involved:**  
- Get the product  
- Add item id (code)  
- Create the invoice  
- Send the invoice  
- Add reminders config  
- Sticky Notes (Get the item, Add item id, Create the invoice, Send the invoice)

**Node Details:**

- **Get the product**  
  - Type: QuickBooks (getAll)  
  - Role: Retrieves product/service item by name from QuickBooks items.  
  - Config: Filter query `WHERE name = '{{ $json.item.name }}'`.  
  - Inputs: JSON with item name from formatted data.  
  - Outputs: Item record(s).  
  - Failures: Item not found, API errors.

- **Add item id**  
  - Type: Code (JavaScript)  
  - Role: Adds the QuickBooks item ID into the workflow data JSON.  
  - Inputs: Item record.  
  - Outputs: JSON with `item.id`.  
  - Failures: Missing item ID.

- **Create the invoice**  
  - Type: QuickBooks (create)  
  - Role: Creates invoice with one line item using the retrieved product for the customer.  
  - Config: Line item includes amount 1, itemId, description "Jotform submission", customerId.  
  - Inputs: JSON with customer and item IDs.  
  - Outputs: Newly created invoice record with invoice ID.  
  - Failures: Invalid customer or item ID, API failures.

- **Send the invoice**  
  - Type: QuickBooks (send)  
  - Role: Sends the created invoice to the customer’s email.  
  - Config: Uses invoice ID and customer email from prior nodes.  
  - Inputs: Invoice ID and customer email.  
  - Outputs: Confirmation of email sent.  
  - Failures: Email sending errors, invalid invoice ID.

- **Add reminders config**  
  - Type: Set  
  - Role: Sets configuration parameters for reminders, including data table ID and intervals (2,3,5 days).  
  - Inputs: Invoice sending confirmation.  
  - Outputs: JSON with reminders config data.  
  - Failures: Missing or incorrect dataTableId breaks reminder tracking.

- **Sticky Notes**: Describe each step in product retrieval and invoice generation.

---

#### 1.4 Invoice Data Persistence

**Overview:**  
Stores the newly created invoice details in a database for subsequent reminder tracking.

**Nodes Involved:**  
- Insert invoice id to DB  
- Sticky Note (Insert Invoice To DB)

**Node Details:**

- **Insert invoice id to DB**  
  - Type: Data Table (CRUD)  
  - Role: Inserts invoice metadata including invoiceId, remainingAmount (balance), currency, remindersSent (initialized to 0).  
  - Config: Maps fields from invoice JSON; uses dataTableId from reminders config.  
  - Inputs: Invoice details after sending.  
  - Outputs: Confirmation of DB insertion.  
  - Failures: Invalid dataTableId, DB connection errors.

---

#### 1.5 Scheduled Reminder Trigger and Configuration

**Overview:**  
Triggers daily at 8 AM to start the reminder process and sets the reminder intervals configuration.

**Nodes Involved:**  
- Schedule reminders trigger  
- Add reminders config  
- If2 (check trigger origin)  
- Sticky Notes (Schedule Trigger, Add Reminders Config, Check Trigger)

**Node Details:**

- **Schedule reminders trigger**  
  - Type: Schedule Trigger  
  - Role: Runs daily at 08:00 to initiate reminders processing.  
  - Config: Fixed trigger hour 8 AM.  
  - Outputs: Empty data triggering downstream nodes.

- **Add reminders config**  
  - (Same as in 1.3)  
  - Role: Ensures reminder intervals and dataTableId are set for reminder logic.

- **If2**  
  - Type: If  
  - Role: Determines if workflow was triggered by schedule trigger or invoice creation trigger.  
  - Condition: Checks `isInvoiceTrigger` boolean.  
  - Outputs:  
    - True: Insert invoice ID to DB (invoice creation path).  
    - False: Get all invoices from DB (reminder path).  
  - Failures: Incorrect trigger identification may cause logical errors.

---

#### 1.6 Invoice Reminder Logic and Dispatch

**Overview:**  
Processes each invoice record to decide whether to send a reminder email now, skip for later, or delete the invoice from tracking if fully paid or reminders exhausted.

**Nodes Involved:**  
- Get Invoices  
- Loop over invoices (split in batches)  
- Get the invoice (QuickBooks)  
- Switch (decision node)  
- Send reminder email (Outlook)  
- Increase sent reminders (update DB)  
- If3 (check reminder count)  
- Delete invoice (DB)  
- Sticky Notes (Get All Invoices, Loop Over Invoices, Get Invoice Details, Send Reminders Logic)

**Node Details:**

- **Get Invoices**  
  - Type: Data Table (get all)  
  - Role: Retrieves all invoice records from the tracking DB.  
  - Config: Uses dataTableId from reminders config.  
  - Outputs: List of invoices with remindersSent and balance info.

- **Loop over invoices**  
  - Type: SplitInBatches  
  - Role: Processes invoices one at a time to avoid API overload.  
  - Inputs: List of invoices from DB.  
  - Outputs: Single invoice JSON per iteration.

- **Get the invoice**  
  - Type: QuickBooks (get)  
  - Role: Retrieves the latest invoice details from QuickBooks to verify payment status.  
  - Inputs: invoiceId from DB record.  
  - Outputs: Invoice details including balance.

- **Switch**  
  - Type: Switch (conditional branches)  
  - Role: Determines appropriate action based on invoice status and reminders sent:  
    - "send now": Invoice balance > 0 and reminder interval matches today’s date  
    - "already paid": Invoice balance == 0  
    - "send later": Default catch-all for other cases  
  - Outputs: Branches to sending email, deleting invoice, or skipping.

- **Send reminder email**  
  - Type: Microsoft Outlook  
  - Role: Sends a reminder email to the customer with invoice details and payment link.  
  - Config: HTML formatted email with invoice number, due date, and a call to action button.  
  - Inputs: Invoice details including customer email.  
  - Outputs: Confirmation of email sent.

- **Increase sent reminders**  
  - Type: Data Table (update)  
  - Role: Updates the invoice record in DB to increment remindersSent and update lastSentAt timestamp.  
  - Inputs: Current remindersSent count.  
  - Outputs: Updated DB record.

- **If3**  
  - Type: If  
  - Role: Checks if remindersSent count exceeds configured intervals length.  
  - Outputs:  
    - True: Delete invoice from DB.  
    - False: Continue looping.

- **Delete invoice**  
  - Type: Data Table (deleteRows)  
  - Role: Deletes invoice record from DB to stop reminder processing.  
  - Inputs: invoice ID.  
  - Outputs: Confirmation of deletion.

- **Sticky Notes**: Explain the reminder decision logic and looping.

---

#### 1.7 AI-Based Summary and Email Report

**Overview:**  
Aggregates today’s sent reminders and uses an AI agent to generate a professional HTML summary email sent to the internal team.

**Nodes Involved:**  
- Get today's sent reminders  
- AI Agent (Langchain Agent)  
- OpenAI Chat Model (GPT-4o-mini)  
- Send reminders sent summary (Outlook)  
- Sticky Notes (Get Sent Reminders, Summarize Sent Reminders & Send An Email)

**Node Details:**

- **Get today's sent reminders**  
  - Type: Data Table (get)  
  - Role: Queries DB for reminders sent on the current day (filter on lastSentAt >= start of day).  
  - Outputs: List of invoice reminder records sent today.

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Uses AI to summarize the reminders list into a professional HTML email content.  
  - Config: System prompt details structure, required fields, and formatting rules for the summary.  
  - Inputs: JSON array of invoices with fields like remainingAmount, currency, remindersSent.  
  - Outputs: HTML summary string.

- **OpenAI Chat Model**  
  - Type: Language Model (OpenAI GPT-4o-mini)  
  - Role: Provides the AI natural language processing for the agent.  
  - Inputs: Prompt from AI Agent.  
  - Outputs: AI-generated summary email content.

- **Send reminders sent summary**  
  - Type: Microsoft Outlook  
  - Role: Sends the generated summary email to internal recipients (e.g., sales@example.com).  
  - Inputs: HTML output from AI Agent node.  
  - Outputs: Email sent confirmation.

- **Sticky Notes**: Provide context on AI summarization and reporting.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                                         | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                                                                                                                                                                                                                                                                                                  |
|---------------------------|-------------------------------|---------------------------------------------------------|--------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Receive form submission   | Webhook                       | Entry point; receives Jotform submissions               | -                              | Format data                     | ## Receive Submission\nReceives the product/service form submission from Jotform                                                                                                                                                                                                                                                                                              |
| Format data              | Code                         | Parses and formats raw submission data                   | Receive form submission         | Check if the customer exists    | ## Format Data\nFormats the data thus making it easier to be used in other nodes                                                                                                                                                                                                                                                                                              |
| Check if the customer exists | QuickBooks (getAll)           | Searches for customer by email in QuickBooks             | Format data                    | If                             | ## Check If Customer exists\nChecks if the customer exists in QBO or not                                                                                                                                                                                                                                                                                                     |
| If                       | If                           | Branch: customer exists? update or create                | Check if the customer exists    | Update the customer / Create the customer | ## Check If Customer exists\nChecks if the customer exists in QBO or not                                                                                                                                                                                                                                                                                                     |
| Update the customer      | QuickBooks (update)           | Updates customer billing details                          | If (true branch)                | Add customer id                 | ## Customer Exists\nNow since the customer exists we will update the customer details like updating the billing details with the new one                                                                                                                                                                                                                                      |
| Create the customer      | QuickBooks (create)           | Creates a new customer in QuickBooks                      | If (false branch)               | Add customer id                 | ## Customer Doesn't Exist\nNow since the customer doesn't exist we will create new customer                                                                                                                                                                                                                                                                                  |
| Add customer id          | Code                         | Adds customer ID into data JSON                           | Update the customer / Create the customer | Get the product               | ## Add Customer Id\nAdds customer id to the data                                                                                                                                                                                                                                                                                                                             |
| Get the product          | QuickBooks (getAll)           | Retrieves product/item by name                            | Add customer id                | Add item id                    | ## Get The Item\nGets the selected product/service from QBO                                                                                                                                                                                                                                                                                                                  |
| Add item id              | Code                         | Adds item ID into data JSON                               | Get the product                | Create the invoice             | ## Add Item Id\nAdds item (service/product) id to the data                                                                                                                                                                                                                                                                                                                   |
| Create the invoice       | QuickBooks (create)           | Creates invoice with item for customer                    | Add item id                   | Send the invoice              | ## Create The Invoice\nCreates a new invoice for that customer                                                                                                                                                                                                                                                                                                               |
| Send the invoice         | QuickBooks (send)             | Emails invoice to customer                                | Create the invoice            | Add reminders config          | ## Send The Invoice\nSends the newly created invoice for that customer(via email)                                                                                                                                                                                                                                                                                             |
| Add reminders config     | Set                          | Sets reminders config data                               | Send the invoice / Schedule reminders trigger | If2                         | ## Add Reminders Config\nAdds reminders config details like `intervals in days` so first reminder will be sent after 2 days, second one after 3 days and final one after 5 days                                                                                                                                                                                             |
| If2                      | If                           | Checks if the trigger origin is invoice creation or schedule | Add reminders config           | Insert invoice id to DB / Get Invoices | ## Check Trigger\nChecks if the previous node has been executed by the above workflow or by the schedule trigger                                                                                                                                                                                                                                                             |
| Insert invoice id to DB  | Data Table (insert)           | Inserts invoice metadata into DB                         | If2 (invoice creation branch)  | -                             | ## Insert Invoice To DB\nInserts newly created invoice needed details to DB so customer will be notified later on about the invoice                                                                                                                                                                                                                                          |
| Schedule reminders trigger | Schedule Trigger              | Daily trigger at 8 AM to start reminders process         | -                              | Add reminders config          | ## Schedule Trigger\nSchedules reminders trigger daily at 8 AM                                                                                                                                                                                                                                                                                                               |
| Get Invoices             | Data Table (getAll)           | Retrieves all invoices for reminders processing           | If2 (schedule branch)          | Loop over invoices            | ## Get All Invoices\nGets all the invoices from DB                                                                                                                                                                                                                                                                                                                           |
| Loop over invoices       | SplitInBatches               | Processes invoices one at a time                          | Get Invoices                   | Get today's sent reminders / Get the invoice | ## Loop Over Invoices\nLoops over invoices one by one                                                                                                                                                                                                                                                                                                                        |
| Get today's sent reminders | Data Table (get)             | Gets reminders sent today from DB                         | Loop over invoices             | AI Agent                     | ## Get Sent Reminders\nGets today's sent reminders from DB                                                                                                                                                                                                                                                                                                                   |
| Get the invoice          | QuickBooks (get)              | Gets latest invoice details from QuickBooks              | Loop over invoices             | Switch                       | ## Get Invoice Details\nGets the invoice details from QBO so we know whether or not any changes have been made or not                                                                                                                                                                                                                                                        |
| Switch                   | Switch                       | Decision: send reminder, skip, or delete invoice         | Get the invoice                | Send reminder email / Delete invoice / Loop over invoices | ## Send Reminders Logic\nThe logic that decides whether or not to send a reminder email now, skip it and send it later or delete the invoice/s from DB (because all the reminders have been sent or the invoice has been paid)                                                                                                                                                 |
| Send reminder email      | Microsoft Outlook             | Sends reminder email to customer                          | Switch (send now branch)       | Increase sent reminders       |                                                                                                                                                                                                                                                                                                                                                                              |
| Increase sent reminders  | Data Table (update)           | Updates remindersSent count and lastSentAt timestamp     | Send reminder email            | If3                         |                                                                                                                                                                                                                                                                                                                                                                              |
| If3                      | If                           | Checks if all reminders have been sent                    | Increase sent reminders        | Delete invoice / Loop over invoices |                                                                                                                                                                                                                                                                                                                                                                              |
| Delete invoice           | Data Table (deleteRows)       | Deletes invoice record from DB                            | If3 (true branch) / Switch (already paid branch) | Loop over invoices            |                                                                                                                                                                                                                                                                                                                                                                              |
| AI Agent                 | Langchain Agent               | AI summary generation for daily reminders                | Get today's sent reminders / OpenAI Chat Model | Send reminders sent summary | ## Summarize Sent Reminders & Send An Email\nSummarizes today's sent reminders using AI and send a summery email to the team like sales team or finance team                                                                                                                                                                                                                 |
| OpenAI Chat Model        | Language Model (OpenAI)       | Provides GPT-4o-mini AI text generation                   | AI Agent                      | AI Agent                    |                                                                                                                                                                                                                                                                                                                                                                              |
| Send reminders sent summary | Microsoft Outlook             | Sends daily summary email to internal team               | AI Agent                      | -                           |                                                                                                                                                                                                                                                                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node: "Receive form submission"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `497e4b6f-7d3d-4839-8acb-68719b65491c`)  
   - Purpose: To receive Jotform submissions.

2. **Add a Code Node: "Format data"**  
   - Language: JavaScript  
   - Purpose: Parse form submission and extract structured customer, address, and item data.  
   - Key code: Use regex to parse billingAddress HTML string; extract customer name, email, phone, and item name.

3. **Add a QuickBooks Node: "Check if the customer exists"**  
   - Resource: Customer  
   - Operation: getAll  
   - Filter Query: `Where PrimaryEmailAddr = '{{ $json.customer.email }}'`  
   - Purpose: Check if customer exists in QuickBooks by email.

4. **Add an If Node: "If"**  
   - Condition: Check if customer record exists and `$json.Id` is present.  
   - True branch → update customer  
   - False branch → create customer

5. **Add QuickBooks Node (true branch): "Update the customer"**  
   - Operation: update  
   - Customer ID: `={{ $json.Id }}`  
   - Update fields: Billing address, GivenName, PrimaryPhone from formatted data.

6. **Add QuickBooks Node (false branch): "Create the customer"**  
   - Operation: create  
   - Fields: GivenName, PrimaryPhone, PrimaryEmailAddr, BillAddr from formatted data.

7. **Add Code Node: "Add customer id"**  
   - Purpose: Add `customer.id` field with the QuickBooks customer ID to data JSON.

8. **Add QuickBooks Node: "Get the product"**  
   - Resource: Item  
   - Operation: getAll  
   - Filter Query: `WHERE name = '{{ $json.item.name }}'`

9. **Add Code Node: "Add item id"**  
   - Purpose: Add `item.id` field with QuickBooks product ID.

10. **Add QuickBooks Node: "Create the invoice"**  
    - Operation: create  
    - Resource: Invoice  
    - Line item: Amount 1, itemId, Description "Jotform submission"  
    - CustomerRef: `={{ $json.customer.id }}`

11. **Add QuickBooks Node: "Send the invoice"**  
    - Operation: send  
    - Invoice ID: `={{ $json.Id }}`  
    - Email: Customer email from customer data.

12. **Add Set Node: "Add reminders config"**  
    - JSON output:  
      ```json
      {
        "dataTableId": "",  // Set your DB table ID here
        "reminderIntervalsInDays": [2, 3, 5],
        "isInvoiceTrigger": true
      }
      ```

13. **Add If Node: "If2"**  
    - Condition: Check if `isInvoiceTrigger` is true  
    - True branch → Insert invoice to DB  
    - False branch → Get all invoices (start reminders process)

14. **Add Data Table Node: "Insert invoice id to DB"**  
    - Operation: insert  
    - Map: invoiceId, remainingAmount, currency, remindersSent=0 from invoice data  
    - Use dataTableId from reminders config.

15. **Add Schedule Trigger Node: "Schedule reminders trigger"**  
    - Trigger daily at 08:00 AM.

16. **Connect "Schedule reminders trigger" to "Add reminders config"**  
    - Set `isInvoiceTrigger` to false in this path.

17. **Add Data Table Node: "Get Invoices"**  
    - Operation: getAll  
    - Use dataTableId from reminders config.

18. **Add SplitInBatches Node: "Loop over invoices"**  
    - Batch size: default (usually 1) to process invoices individually.

19. **Add QuickBooks Node: "Get the invoice"**  
    - Operation: get  
    - Invoice ID from DB record.

20. **Add Switch Node: "Switch"**  
    - Conditions:  
      - send now: `Balance > 0 && next reminder date == today`  
      - already paid: `Balance == 0`  
      - send later: default

21. **Add Microsoft Outlook Node: "Send reminder email"**  
    - Subject: Friendly reminder with invoice number.  
    - To: Customer billing email.  
    - Body: Styled HTML content with invoice info and payment link.

22. **Add Data Table Node: "Increase sent reminders"**  
    - Operation: update  
    - Increment `remindersSent` by 1  
    - Update `lastSentAt` with current timestamp.

23. **Add If Node: "If3"**  
    - Condition: Check if `remindersSent >= length of reminder intervals`  
    - True → Delete invoice from DB  
    - False → Continue looping.

24. **Add Data Table Node: "Delete invoice"**  
    - Operation: deleteRows by invoiceId.

25. **Add Data Table Node: "Get today's sent reminders"**  
    - Operation: get  
    - Filter: `lastSentAt >= start of current day`.

26. **Add Langchain Agent Node: "AI Agent"**  
    - System message: instruct summarization and HTML formatting of reminders sent today.  
    - Input: Today's reminders data.

27. **Add OpenAI Chat Model Node: "OpenAI Chat Model"**  
    - Model: gpt-4o-mini  
    - Connect input from AI Agent prompt.

28. **Connect OpenAI Chat Model output back to AI Agent**  
    - To process the AI response.

29. **Add Microsoft Outlook Node: "Send reminders sent summary"**  
    - Subject: Summary of today's reminders sent  
    - To: Internal team email (e.g., sales@example.com)  
    - Body: AI-generated HTML summary.

30. **Connect all nodes according to the data flow described in the overview**  
    - Pay attention to branching (If nodes), batches, and trigger conditions.

31. **Configure Credentials:**  
    - QuickBooks OAuth2 credentials with appropriate scopes (customers, invoices).  
    - Outlook OAuth2 credentials for sending email.  
    - OpenAI API credentials for Langchain nodes.

32. **Create Database Table for Invoice Tracking:**  
    - Columns: invoiceId (string), remainingAmount (number), currency (string), remindersSent (number), lastSentAt (datetime).  
    - Provide dataTableId to Set node "Add reminders config".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Full workflow automates invoice creation and reminders integrating Jotform, QuickBooks Online, and Outlook email with AI-based daily summary. Suitable for freelancers, service providers, small businesses, and e-commerce sellers.                                                                                                                                                                                                                                                                                                                                                                                                | Workflow description and use case                                                                                                                                                                                               |
| Requires Jotform webhook setup: https://www.jotform.com/help/245-how-to-setup-a-webhook-with-jotform/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Jotform webhook setup instructions                                                                                                                                                                                             |
| QuickBooks Online API setup and credentials needed: https://developer.intuit.com/app/developer/qbo/docs/get-started/get-client-id-and-client-secret                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | QuickBooks Online developer documentation                                                                                                                                                                                      |
| Outlook node configuration guide for email sending: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Microsoft Outlook integration documentation                                                                                                                                                                                    |
| Database table schema is critical for tracking invoice reminder status; ensure columns: invoiceId (string), remainingAmount (number), currency (string), remindersSent (number), lastSentAt (datetime). Reminder intervals can be customized in "Add reminders config" node.                                                                                                                                                                                                                                                                                                                                                                   | Database setup requirement                                                                                                                                                                                                     |
| AI summary uses Langchain agent with OpenAI GPT-4o-mini; prompt designed for professional HTML email summaries with invoice details and reminders count.                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | AI integration details and prompt template                                                                                                                                                                                    |
| Sticky notes throughout the workflow provide context and explanations for each logical block; reviewing them aids understanding and maintenance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Visual documentation embedded in workflow                                                                                                                                                                                     |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow implemented with n8n, a no-code integration and automation platform. All processing complies strictly with current content policies and contains no illegal, offensive, or protected content. All handled data is lawful and publicly accessible.