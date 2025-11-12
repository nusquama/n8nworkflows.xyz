Generate Invoices and Send AI-Powered Reminders with Jotform and Xero

https://n8nworkflows.xyz/workflows/generate-invoices-and-send-ai-powered-reminders-with-jotform-and-xero-9800


# Generate Invoices and Send AI-Powered Reminders with Jotform and Xero

### 1. Workflow Overview

This workflow automates invoice generation and customer payment reminders by integrating **Jotform** (form submissions), **Xero** (contact and invoice management), and **email services** enhanced by AI-generated content. It caters to freelancers, service providers, small businesses, and e-commerce sellers who need seamless invoicing and follow-up.

**Logical Blocks:**

- **1.1 Receive Submission**  
  Accepts product/service orders from a Jotform webhook, extracting and formatting customer and order data.

- **1.2 Create/Update Contact in Xero**  
  Creates or updates the customer record in Xero based on the submission.

- **1.3 Create Invoice in Xero**  
  Generates an invoice for the customer with specified product/service details.

- **1.4 Send Invoice Email via AI-generated Content**  
  Uses AI to create a professional invoice email, then sends it to the customer.

- **1.5 Store Invoice Details in Data Table**  
  Saves invoice metadata into a data table for tracking reminders.

- **1.6 Scheduled Reminder Trigger**  
  Daily schedule triggers the reminder logic to assess outstanding invoices.

- **1.7 Retrieve and Loop Over Invoices**  
  Fetches all stored invoices, iterates through each for reminder evaluation.

- **1.8 Reminder Decision Logic**  
  Determines whether to send a reminder now, delay it, or remove the invoice if paid or reminders exhausted.

- **1.9 Send Reminder Emails with AI-generated Content**  
  Sends reminder emails crafted by AI.

- **1.10 Update Reminder Tracking and Cleanup**  
  Updates reminder counts and deletes invoices from tracking as needed.

- **1.11 Summarize Sent Reminders and Email Summary**  
  Generates and emails a daily summary of reminders sent to the internal team using AI.

---

### 2. Block-by-Block Analysis

#### 1.1 Receive Submission

- **Overview:**  
  This initial block receives incoming form submissions from Jotform via webhook, capturing customer and order data.

- **Nodes Involved:**  
  - Receive form submission (Webhook)  
  - Format data (Code)  
  - Sticky Notes: "Receive Submission", "Format Data"

- **Node Details:**  
  - *Receive form submission*  
    - Type: Webhook  
    - Config: POST endpoint triggered by Jotform form webhook  
    - Input: HTTP POST with form data (name, email, phone, itemName, billingAddress)  
    - Output: Raw JSON payload  
    - Potential failures: webhook timeout, malformed payload

  - *Format data*  
    - Type: Code (JavaScript)  
    - Config: Extracts and parses billing address text into structured address fields; formats customer and item info  
    - Expressions: Uses `$input.first().json.body` for form fields, regex for address parsing  
    - Output: JSON object with structured `address`, `customer`, and `item` fields  
    - Edge cases: Address field format mismatch causing failed regex match

#### 1.2 Create/Update Contact in Xero

- **Overview:**  
  Creates or updates customer contact in Xero using data from formatted submission.

- **Nodes Involved:**  
  - Create/Update the contact (Xero)  
  - Sticky Note: "Create/Update Contact"

- **Node Details:**  
  - Type: Xero Contact resource node  
  - Config: Uses customer name, email, phone, and structured address from previous node  
  - Expressions: `={{ $json.customer.name }}`, `={{ $json.address.city }}`, etc.  
  - Input: Output from Format data  
  - Output: Xero contact response including ContactID  
  - Failures: Authentication errors, API rate limits, missing required fields

#### 1.3 Create Invoice in Xero

- **Overview:**  
  Generates a new invoice for the contact with specified item details.

- **Nodes Involved:**  
  - Create the invoice (Xero)  
  - Sticky Note: "Create The Invoice"

- **Node Details:**  
  - Type: Xero Invoice resource node  
  - Config: Invoice type ACCREC (accounts receivable), contactId from previous node, line item includes itemCode matching `item.name` from formatted data, fixed unit amount 10, tax type INPUT  
  - Expressions: `={{ $json.ContactID }}`, `={{ $('Format data').item.json.item.name }}`  
  - Output: Created invoice details (InvoiceID, AmountDue, Contact Email)  
  - Failures: Item code mismatch with Xero catalog, invalid contactId

#### 1.4 Send Invoice Email via AI-generated Content

- **Overview:**  
  Uses AI to generate professional HTML email content for the invoice and sends it to the customer.

- **Nodes Involved:**  
  - AI Agent (Langchain Agent)  
  - OpenAI Chat Model (GPT-4o-mini)  
  - Send email (SMTP)  
  - Sticky Note: "Send The Invoice"

- **Node Details:**  
  - *AI Agent*  
    - Type: Langchain AI Agent  
    - Config: Receives Xero invoice JSON, system message prompts to generate professional invoice email HTML  
    - Output: HTML email content  
    - Failures: API quota limits, invalid AI prompt or response

  - *OpenAI Chat Model*  
    - Type: Langchain OpenAI Chat node  
    - Config: Model `gpt-4o-mini`  
    - Credential: OpenAI API key  
    - Connection: AI Agent requests model for generation

  - *Send email*  
    - Type: Email Send  
    - Config: Sends email using SMTP with generated HTML content, subject "New Invoice", to customer email from invoice data  
    - Credential: SMTP (Mailtrap)  
    - Failures: SMTP auth errors, invalid recipient email

#### 1.5 Store Invoice Details in Data Table

- **Overview:**  
  Inserts invoice metadata into a data table for later reminder tracking.

- **Nodes Involved:**  
  - Insert invoice id to DB (Data Table)  
  - Add reminders config (Set node)  
  - Sticky Note: "Insert Invoice To DB", "Add Reminders Config"

- **Node Details:**  
  - *Add reminders config*  
    - Type: Set node  
    - Config: Sets `dataTableId`, `reminderIntervalsInDays` ([2,3,5]), and flag `isInvoiceTrigger` to identify trigger source  
    - Input: From invoice creation chain or schedule trigger  
    - Output: Configuration JSON

  - *Insert invoice id to DB*  
    - Type: Data Table (n8n internal DB)  
    - Config: Inserts invoiceId, remainingAmount, currency, remindersSent=0  
    - DataTableId dynamically set by Add reminders config node  
    - Failures: DB connectivity, duplicate insertions

#### 1.6 Scheduled Reminder Trigger

- **Overview:**  
  Triggers daily at 8 AM to start the reminder process.

- **Nodes Involved:**  
  - Schedule reminders trigger (Schedule Trigger)  
  - Sticky Note: "Schedule Trigger"

- **Node Details:**  
  - Type: Schedule Trigger  
  - Config: Triggers daily at 08:00 hour  
  - Output: Trigger event to start reminder flow

#### 1.7 Retrieve and Loop Over Invoices

- **Overview:**  
  Retrieves all invoices from the Data Table and processes them one-by-one.

- **Nodes Involved:**  
  - Get Invoices (Data Table)  
  - Loop over invoices (Split in Batches)  
  - Sticky Notes: "Get All Invoices", "Loop Over Invoices"

- **Node Details:**  
  - *Get Invoices*  
    - Type: Data Table get all rows  
    - Config: Returns all invoice entries from configured data table  
    - Failures: DB read errors

  - *Loop over invoices*  
    - Type: SplitInBatches  
    - Config: Processes invoices individually (batch size 1)  
    - Output: Single invoice item per batch

#### 1.8 Reminder Decision Logic

- **Overview:**  
  For each invoice, checks if a reminder should be sent now, later, or if invoice is paid/all reminders sent (thus delete).

- **Nodes Involved:**  
  - Get today's sent reminders (Data Table)  
  - Get the invoice (Xero)  
  - Switch (Decision node)  
  - If2 & If3 (Conditional nodes)  
  - Sticky Notes: "Get Sent Reminders", "Get Invoice Details", "Send Reminders Logic", "Check Trigger"

- **Node Details:**  
  - *Get today's sent reminders*  
    - Filters reminders sent with lastSentAt >= start of current day  
    - Executes once per invoice iteration

  - *Get the invoice*  
    - Fetches invoice details from Xero by invoiceId for updated status and amount due

  - *Switch*  
    - Conditions:  
      - "send now": If invoice amount due > 0 AND today matches reminder interval based on last sent date + configured intervals  
      - "already paid": If amount due == 0  
      - "send later": Default fallback  

  - *If2*  
    - Checks trigger source is invoice creation (isInvoiceTrigger true)

  - *If3*  
    - Checks if remindersSent count >= configured max intervals, if true triggers deletion

- **Potential issues:** Incorrect date calculations, missing invoice data, API failures

#### 1.9 Send Reminder Emails with AI-generated Content

- **Overview:**  
  Sends reminder emails to customers using AI-generated professional content.

- **Nodes Involved:**  
  - Send reminder email (Email Send)  
  - Increase sent reminders (Data Table update)  
  - Sticky Note: none directly, part of reminder logic flow

- **Node Details:**  
  - *Send reminder email*  
    - Sends HTML email with invoice reminder details, subject includes invoice number  
    - SMTP credentials (Mailtrap)  
    - Template includes styled HTML with placeholders for invoice data

  - *Increase sent reminders*  
    - Updates data table incrementing remindersSent count and lastSentAt timestamp for invoice

- **Failures:** SMTP delivery errors, data update conflicts

#### 1.10 Update Reminder Tracking and Cleanup

- **Overview:**  
  Deletes invoice entries from data table if paid or reminders exhausted.

- **Nodes Involved:**  
  - Delete invoice (Data Table)  
  - Connected from If3 and Switch nodes

- **Node Details:**  
  - Deletes invoice row identified by invoiceId from tracking data table

- **Failure modes:** Data Table delete errors, race conditions

#### 1.11 Summarize Sent Reminders and Email Summary

- **Overview:**  
  Summarizes reminders sent today using AI and emails a report to internal team.

- **Nodes Involved:**  
  - AI Agent1 (Langchain Agent)  
  - OpenAI Chat Model1 (GPT-4o-mini)  
  - Send reminders sent summary (Email Send)  
  - Sticky Note: "Summarize Sent Reminders & Send An Email"

- **Node Details:**  
  - *AI Agent1*  
    - Receives list of reminders sent today, generates HTML summary email addressed to sales/finance team  
    - System prompt guides format and content

  - *OpenAI Chat Model1*  
    - Chat model used for summary generation

  - *Send reminders sent summary*  
    - Sends email with AI-generated summary to internal email (sales@example.com)  
    - SMTP credentials Mailtrap

- **Potential failures:** AI API quota, malformed data input, SMTP issues

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                        | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                      |
|---------------------------|-----------------------|-------------------------------------|--------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| Receive form submission    | Webhook               | Receives Jotform form submission     | —                              | Format data                   | ## Receive Submission Receives the product/service form submission from Jotform                 |
| Format data               | Code                  | Extracts and formats submission data | Receive form submission         | Create/Update the contact      | ## Format Data Formats the data thus making it easier to be used in other nodes                 |
| Create/Update the contact  | Xero                  | Creates or updates Xero contact      | Format data                    | Create the invoice             | ## Create/Update Contact Creates or updates the contact                                         |
| Create the invoice         | Xero                  | Generates invoice for contact        | Create/Update the contact       | AI Agent, Add reminders config | ## Create The Invoice Creates a new invoice for that contact                                    |
| AI Agent                  | Langchain Agent       | Generates professional invoice email | Create the invoice             | Send email                    | ## Send The Invoice Sends the newly created invoice for that customer(via email)                |
| OpenAI Chat Model          | Langchain LM Chat     | AI model for email content generation | AI Agent                      | AI Agent                      |                                                                                                 |
| Send email                | Email Send            | Sends invoice email to customer      | AI Agent                      | —                             |                                                                                                 |
| Add reminders config       | Set                   | Sets reminder intervals and config   | Create the invoice             | If2                           | ## Add Reminders Config Adds reminders config details like `intervals in days`                  |
| If2                       | If                    | Checks trigger source is invoice creation | Add reminders config           | Insert invoice id to DB / Get Invoices |                                                                                                 |
| Insert invoice id to DB    | Data Table            | Stores invoice details for reminders | If2                           | —                             | ## Insert Invoice To DB Inserts newly created invoice needed details to DB                      |
| Schedule reminders trigger | Schedule Trigger      | Triggers daily reminder process      | —                              | Add reminders config           | ## Schedule Trigger Schedules reminders trigger daily at 8 AM                                  |
| Get Invoices              | Data Table            | Retrieves all invoices from DB       | If2                           | Loop over invoices             | ## Get All Invoices Gets all the invoices from DB                                              |
| Loop over invoices         | SplitInBatches        | Iterates over each invoice            | Get Invoices, Delete invoice, Switch | Get today's sent reminders, Get the invoice | ## Loop Over Invoices Loops over invoices one by one                                          |
| Get today's sent reminders | Data Table            | Retrieves reminders sent today        | Loop over invoices             | AI Agent1                     | ## Get Sent Reminders Gets today's sent reminders from DB                                     |
| Get the invoice            | Xero                  | Gets invoice details from Xero       | Loop over invoices             | Switch                       | ## Get Invoice Details Gets the invoice details from Xero so we know whether or not any changes have been made |
| Switch                    | Switch                | Decides reminder action                | Get the invoice                | Send reminder email / Delete invoice / Loop over invoices | ## Send Reminders Logic The logic that decides whether or not to send a reminder email now, skip, or delete |
| Send reminder email        | Email Send            | Sends invoice reminder email          | Switch                        | Increase sent reminders        |                                                                                                 |
| Increase sent reminders    | Data Table            | Updates reminder count and timestamp  | Send reminder email            | If3                           |                                                                                                 |
| If3                       | If                    | Checks if max reminders sent to delete | Increase sent reminders         | Delete invoice / Loop over invoices |                                                                                                 |
| Delete invoice             | Data Table            | Deletes invoice from reminder tracking | If3, Switch                   | Loop over invoices             |                                                                                                 |
| AI Agent1                 | Langchain Agent       | Generates daily reminder summary email | Get today's sent reminders     | Send reminders sent summary    | ## Summarize Sent Reminders & Send An Email Summarizes today's sent reminders using AI         |
| OpenAI Chat Model1         | Langchain LM Chat     | AI model for summary email generation | AI Agent1                     | AI Agent1                     |                                                                                                 |
| Send reminders sent summary| Email Send            | Sends daily reminder summary to team  | AI Agent1                     | —                             |                                                                                                 |
| Sticky Notes (various)     | Sticky Note           | Documentation and explanation nodes  | —                              | —                             | See detailed notes per node in section 2                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: "Receive form submission"  
   - HTTP Method: POST  
   - Path: Unique string (e.g., "148f97af-2d29-4836-a910-2c77b7c33b26")  
   - Purpose: Receive Jotform submissions

2. **Add Code Node to Format Data**  
   - Type: Code  
   - Name: "Format data"  
   - JavaScript: Parse billingAddress string into fields; extract name, email, phone, itemName  
   - Connect input from webhook node

3. **Add Xero Contact Node**  
   - Type: Xero (Contact resource)  
   - Name: "Create/Update the contact"  
   - Parameters: Fill name, emailAddress, phone, and address fields using expressions referencing previous node JSON  
   - Credentials: Xero OAuth2 API credentials configured  
   - Connect input from Format data node

4. **Add Xero Invoice Node**  
   - Type: Xero (Invoice resource)  
   - Name: "Create the invoice"  
   - Parameters:  
     - Type: ACCREC  
     - contactId: from previous contact node output  
     - lineItems: itemCode from formatted data, unitAmount fixed at 10, taxType INPUT, accountCode 200  
   - Credentials: Xero OAuth2 API  
   - Connect input from contact node

5. **Add Langchain AI Agent Node**  
   - Type: Langchain Agent  
   - Name: "AI Agent"  
   - Parameters: Pass invoice JSON, system message to generate professional invoice email HTML  
   - Connect input from "Create the invoice" node

6. **Add Langchain OpenAI Chat Model Node**  
   - Type: Langchain LM Chat OpenAI  
   - Name: "OpenAI Chat Model"  
   - Model: gpt-4o-mini  
   - Credentials: OpenAI API key  
   - Connect AI Agent to this node as language model

7. **Add Email Send Node**  
   - Type: Email Send  
   - Name: "Send email"  
   - Parameters:  
     - Subject: "New Invoice"  
     - To: Customer email from invoice data  
     - From: system@example.com (replace as needed)  
     - HTML Body: Output from AI Agent  
   - Credentials: SMTP (Mailtrap or other)  
   - Connect input from AI Agent

8. **Add Set Node for Reminders Config**  
   - Type: Set  
   - Name: "Add reminders config"  
   - Parameters:  
     - dataTableId: Your data table ID for invoice reminders  
     - reminderIntervalsInDays: [2, 3, 5] (customizable)  
     - isInvoiceTrigger: true if triggered by invoice creation, false if schedule trigger  
   - Connect input from "Create the invoice" node and from schedule trigger node (see below)

9. **Add If Node (If2) to Check Trigger Source**  
   - Type: If  
   - Name: "If2"  
   - Condition: isInvoiceTrigger == true  
   - True branch: Insert invoice to DB  
   - False branch: Get invoices for reminders

10. **Add Data Table Node to Insert Invoice Metadata**  
    - Type: Data Table  
    - Name: "Insert invoice id to DB"  
    - Parameters: Insert invoiceId, remainingAmount, currency, remindersSent=0  
    - DataTableId from reminders config node  
    - Connect True branch from If2

11. **Add Data Table Node to Get All Invoices**  
    - Type: Data Table  
    - Name: "Get Invoices"  
    - Parameters: Get all rows from data table  
    - DataTableId from reminders config node  
    - Connect False branch from If2

12. **Add SplitInBatches Node**  
    - Type: SplitInBatches  
    - Name: "Loop over invoices"  
    - Batch size: 1  
    - Connect input from Get Invoices

13. **Add Data Table Node to Get Today's Sent Reminders**  
    - Type: Data Table  
    - Name: "Get today's sent reminders"  
    - Parameters: Filter on lastSentAt >= start of current day  
    - DataTableId from reminders config  
    - Connect input from Loop over invoices

14. **Add Xero Invoice Get Node**  
    - Type: Xero (Invoice resource, operation get)  
    - Name: "Get the invoice"  
    - Parameters: invoiceId from current invoice JSON  
    - Credentials: Xero OAuth2 API  
    - Connect input from Loop over invoices

15. **Add Switch Node to Decide Reminder Action**  
    - Type: Switch  
    - Name: "Switch"  
    - Conditions:  
      - send now: AmountDue > 0 AND reminder interval matches today  
      - already paid: AmountDue == 0  
      - send later: default  
    - Connect input from Get the invoice

16. **Add Email Send Node for Reminder Email**  
    - Type: Email Send  
    - Name: "Send reminder email"  
    - Parameters: HTML reminder template with invoice info, subject includes invoice number  
    - Credentials: SMTP  
    - Connect from Switch "send now" output

17. **Add Data Table Node to Increase Sent Reminders**  
    - Type: Data Table Update  
    - Name: "Increase sent reminders"  
    - Updates remindersSent count + 1 and lastSentAt timestamp  
    - Connect input from Send reminder email

18. **Add If Node (If3) to Check Max Reminders Sent**  
    - Type: If  
    - Name: "If3"  
    - Condition: remindersSent >= max intervals count (e.g., 3)  
    - True: Delete invoice from tracking  
    - False: Loop back to next invoice  
    - Connect input from Increase sent reminders

19. **Add Data Table Node to Delete Invoice**  
    - Type: Data Table Delete  
    - Name: "Delete invoice"  
    - Parameters: Delete by invoiceId  
    - Connect from If3 True and Switch "already paid" outputs

20. **Add Schedule Trigger Node**  
    - Type: Schedule Trigger  
    - Name: "Schedule reminders trigger"  
    - Parameters: Daily at 8 AM  
    - Connect output to Add reminders config node with isInvoiceTrigger false

21. **Add Langchain AI Agent Node for Summary**  
    - Type: Langchain Agent  
    - Name: "AI Agent1"  
    - Parameters: Receives list of reminders sent today, generates HTML summary email  
    - Connect input from Get today's sent reminders

22. **Add Langchain OpenAI Chat Model Node for Summary**  
    - Type: Langchain LM Chat OpenAI  
    - Name: "OpenAI Chat Model1"  
    - Model: gpt-4o-mini  
    - Credentials: OpenAI API key  
    - Connect AI Agent1 to this node as model

23. **Add Email Send Node for Summary Email**  
    - Type: Email Send  
    - Name: "Send reminders sent summary"  
    - Parameters: Sends summary email to internal team (e.g., sales@example.com)  
    - Credentials: SMTP  
    - Connect input from AI Agent1

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow requires Jotform webhook setup. See: https://www.jotform.com/help/245-how-to-setup-a-webhook-with-jotform/ | Jotform integration instructions |
| Xero credentials setup info: https://docs.n8n.io/integrations/builtin/credentials/xero | Xero OAuth2 credential configuration |
| Ensure product/service names in Jotform match exactly the item `Code` in Xero for invoice creation | Data consistency requirement |
| Email nodes must be updated with valid SMTP credentials and from/to emails | Email delivery setup |
| Data Table schema: invoiceId (string), remainingAmount (number), currency (string), remindersSent (number), lastSentAt (datetime) | Database tracking schema |
| Reminder intervals configurable in "Add reminders config" node as [2,3,5] days by default | Reminder schedule customization |
| AI agents require valid LLM model credentials (OpenAI API key) | AI content generation setup |

---

This structured documentation enables understanding, modification, and reproduction of the full workflow including its logic, node configuration, dependencies, and integration points with external services.