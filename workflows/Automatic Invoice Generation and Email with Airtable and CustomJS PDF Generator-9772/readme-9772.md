Automatic Invoice Generation and Email with Airtable and CustomJS PDF Generator

https://n8nworkflows.xyz/workflows/automatic-invoice-generation-and-email-with-airtable-and-customjs-pdf-generator-9772


# Automatic Invoice Generation and Email with Airtable and CustomJS PDF Generator

### 1. Workflow Overview

This workflow automates the generation and emailing of invoices based on data stored in Airtable. It targets businesses that manage invoices and client data in Airtable and want to streamline invoicing by automatically generating PDF invoices and sending them via email. The workflow retrieves invoices marked as “Ready”, gathers associated invoice items and client details, generates a PDF invoice using a custom JavaScript PDF generator, sends the invoice by email, and finally updates the invoice status to “Sent” to avoid duplicate processing.

**Logical Blocks:**

- **1.1 Trigger and Invoice Retrieval:** Starting point and fetching invoices ready for processing.
- **1.2 Invoice Items Loop and Aggregation:** Iterating over invoices to get their individual invoice items and aggregate them.
- **1.3 Client Data Retrieval and Company Setup:** Fetching client details for each invoice and setting static company details.
- **1.4 Invoice Generation and Email Sending:** Creating the PDF invoice and emailing it.
- **1.5 Invoice Status Update:** Marking invoices as sent to prevent reprocessing.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Invoice Retrieval

**Overview:**  
This block starts the workflow manually and fetches all invoices from Airtable that have the status “Ready” to process them for invoicing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get Ready Invoices  
- Sticky Note (explanatory)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow for testing or controlled execution.  
  - Config: No parameters needed.  
  - Input: None  
  - Output: Triggers next node.  
  - Edge cases: None typical; manual trigger.

- **Get Ready Invoices**  
  - Type: Airtable (Search operation)  
  - Role: Retrieves invoice records where the “Status” field equals “Ready” from the specified Airtable base/table.  
  - Config: Filter by formula `{Status} = 'Ready'` to limit records.  
  - Credentials: Airtable API token required (configured).  
  - Input: Trigger output  
  - Output: List of invoice records with status “Ready”.  
  - Edge cases: API rate limits, missing or invalid credentials, empty result sets.

- **Sticky Note**  
  - Type: Informational  
  - Content: Link to example Airtable base & screenshot for context.

---

#### 1.2 Invoice Items Loop and Aggregation

**Overview:**  
Invoices often have multiple line items. This block loops through each invoice, fetches associated invoice items, maps item fields, and aggregates the items so that each invoice has a consolidated list of items.

**Nodes Involved:**  
- Loop Over Items  
- Get Invoice Items  
- Map Fields  
- Aggregate  
- Sticky Note (explaining loop necessity)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each invoice item batch to process individually.  
  - Config: Default batch size (not explicitly set), processes invoices one by one.  
  - Input: Output of “Get Ready Invoices”  
  - Output: Passes individual invoice data to next nodes.  
  - Edge cases: Large batch sizes may cause timeout; empty batches.

- **Get Invoice Items**  
  - Type: Airtable (Search)  
  - Role: Fetches invoice items linked to the current invoice ID using a formula filter.  
  - Config: Filter formula `=FIND("{{ $json.ID }}", ARRAYJOIN({Invoice}))` to find matching items.  
  - Input: From Loop Over Items  
  - Output: Items related to the current invoice.  
  - Edge cases: No items found, API errors.

- **Map Fields**  
  - Type: Set  
  - Role: Transforms invoice item data to the format required by the invoice generator (description, quantity, unit price, invoiceId).  
  - Config: Uses expressions to extract fields and fallbacks (e.g., custom hourly rate or default hourly rate).  
  - Input: Invoice Items  
  - Output: Structured item data for aggregation.  
  - Edge cases: Missing fields, null values.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all item data into a single “items” array for each invoice.  
  - Config: Aggregates all items from previous Set node outputs.  
  - Input: Output of Map Fields  
  - Output: One consolidated invoice record with items.  
  - Edge cases: Empty item lists.

- **Sticky Note**  
  - Explains the need for looping and aggregation due to mismatch between invoices count and items count.

---

#### 1.3 Client Data Retrieval and Company Setup

**Overview:**  
Fetches client details for each invoice and sets company details statically, preparing data for the invoice generation.

**Nodes Involved:**  
- Get Clients  
- Set Company Details  
- Sticky Note (explaining company details)

**Node Details:**

- **Get Clients**  
  - Type: Airtable (Get by ID)  
  - Role: Retrieves client details based on the “Client ID” from the current invoice.  
  - Config: Uses expression `={{ $('Get Ready Invoices').item.json['Client ID'][0] }}` to get client record ID.  
  - Input: From Loop Over Items (after aggregation)  
  - Output: Client details JSON.  
  - Edge cases: Missing client ID, invalid references.

- **Set Company Details**  
  - Type: Set  
  - Role: Defines static company details such as name, address, tax ID, email, phone, logo URL, and bank details.  
  - Config: Hardcoded values assigned to variables like CompanyName, Address, TaxId, Email, Phone, Logo, Account Number, BIC, Bank Name.  
  - Input: From Get Clients  
  - Output: Merged data passed to invoice generator.  
  - Edge cases: None typical (static data).

- **Sticky Note1**  
  - Indicates where to customize company details.

---

#### 1.4 Invoice Generation and Email Sending

**Overview:**  
Uses a custom JavaScript PDF toolkit node to generate the invoice PDF, then sends it via email with the generated PDF attached.

**Nodes Involved:**  
- Generate Invoice  
- Send Email With Attachment  
- Sticky Note (explaining generation and emailing process)

**Node Details:**

- **Generate Invoice**  
  - Type: Custom JS PDF Toolkit (invoiceGenerator)  
  - Role: Produces PDF invoices using issuer, billing, payment, item, and recipient data.  
  - Config:  
    - Issuer: Fields from Set Company Details node (email, phone, tax ID, address, logo, company name).  
    - Billing: Notes, tax rate (19%), currency (EUR), invoice date, invoice number from “Get Ready Invoices”.  
    - Payment: Bank details from Set Company Details.  
    - Items: Passed as JSON array from aggregated invoice items.  
    - Recipient: Client data from Get Clients node.  
  - Credentials: Uses Custom JS API credentials for PDF generation.  
  - Input: From Set Company Details  
  - Output: PDF data for sending.  
  - Edge cases: Missing or invalid data fields, PDF generation errors.

- **Send Email With Attachment**  
  - Type: Email Send  
  - Role: Sends the PDF invoice as an email attachment to the client.  
  - Config:  
    - Subject: “Your Invoice for Last Month”  
    - To: Hardcoded “info@yourcomp.org” (should be dynamic ideally)  
    - From: Uses `$json.InvoiceEmail` (dynamic from invoice data)  
    - Body text: Standard polite message.  
    - Attachment: PDF data from Generate Invoice node.  
  - Credentials: SMTP account configured for sending email.  
  - Input: From Generate Invoice  
  - Output: Email sent status.  
  - Edge cases: Email sending failure, invalid email addresses, SMTP auth errors.

- **Sticky Note4**  
  - Highlights generation and email sending step.

---

#### 1.5 Invoice Status Update

**Overview:**  
Marks invoices as “Sent” in Airtable after successful email dispatch to prevent duplicate invoicing.

**Nodes Involved:**  
- Update record  
- Sticky Note (explaining status update)

**Node Details:**

- **Update record**  
  - Type: Airtable (Update operation)  
  - Role: Updates invoice record status field to “Sent” based on invoice ID.  
  - Config:  
    - Matching on invoice ID from “Get Ready Invoices” node.  
    - Sets “Status” to “Sent”.  
  - Credentials: Airtable API token  
  - Input: From Send Email With Attachment  
  - Output: Confirmation of update  
  - Edge cases: Update failures, record locking, API errors.

- **Sticky Note2**  
  - Explains marking invoices as sent to avoid reprocessing.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                                | Input Node(s)                     | Output Node(s)                         | Sticky Note                                            |
|---------------------------|-------------------------------|------------------------------------------------|----------------------------------|--------------------------------------|--------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                | Workflow start trigger                          | None                             | Get Ready Invoices                   | Run this workflow manually                              |
| Get Ready Invoices        | Airtable (Search)             | Fetch invoices with Status = “Ready”           | When clicking ‘Execute workflow’ | Loop Over Items                     | Collect Invoice Data From Airtable                      |
| Loop Over Items           | SplitInBatches                | Loop over each invoice to process items        | Get Ready Invoices               | Get Clients, Get Invoice Items       | Loop Over Items                                         |
| Get Invoice Items         | Airtable (Search)             | Get items linked to the current invoice        | Loop Over Items                  | Map Fields                         | Loop Over Items                                         |
| Map Fields                | Set                          | Map invoice item fields for invoice generation | Get Invoice Items                | Aggregate                         | Loop Over Items                                         |
| Aggregate                 | Aggregate                    | Combine invoice items into single list          | Map Fields                      | Loop Over Items                    | Loop Over Items                                         |
| Get Clients               | Airtable (Get by ID)          | Get client details for current invoice          | Loop Over Items                  | Set Company Details                 | Define Your Company Details                             |
| Set Company Details       | Set                          | Set static company information                   | Get Clients                     | Generate Invoice                   | Define Your Company Details                             |
| Generate Invoice          | Custom JS PDF Toolkit         | Generate PDF invoice document                    | Set Company Details             | Send Email With Attachment          | Generate Invoice & send email                           |
| Send Email With Attachment| Email Send                   | Email invoice PDF to client                       | Generate Invoice                | Update record                     | Generate Invoice & send email                           |
| Update record             | Airtable (Update)             | Mark invoice as Sent                              | Send Email With Attachment      | None                             | Mark Invoices as "Sent"                                 |
| Sticky Note               | Sticky Note                  | Informational notes                               | None                           | None                             | Various, see sticky note content                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Add Airtable Node “Get Ready Invoices”**  
   - Operation: Search  
   - Base: Select your Airtable base with invoices.  
   - Table: Select invoice table.  
   - Filter Formula: `{Status} = 'Ready'`  
   - Credentials: Configure Airtable API token with read rights.

3. **Add SplitInBatches Node “Loop Over Items”**  
   - Connect from “Get Ready Invoices”  
   - Use default batch size to process invoices one by one.

4. **Add Airtable Node “Get Invoice Items”**  
   - Operation: Search  
   - Base and Table: Invoice items table.  
   - Filter Formula: `=FIND("{{ $json.ID }}", ARRAYJOIN({Invoice}))` (matches current invoice ID).  
   - Credentials: Same Airtable API token.

5. **Add Set Node “Map Fields”**  
   - Map fields to:  
     - description: `{{$json.Description}}`  
     - quantity: `{{$json.Hours}}`  
     - unitPrice: `{{$json['Custom Hourly Rate'] || $json['Default Hourly Rate'][0]}}`  
     - invoiceId: `{{$json.ID}}`

6. **Add Aggregate Node “Aggregate”**  
   - Aggregate all items into one array field named “items”.

7. **Connect Aggregate output back to “Loop Over Items” node**  
   - To continue processing after aggregation.

8. **Add Airtable Node “Get Clients”**  
   - Operation: Get by ID  
   - Base and Table: Clients table.  
   - ID: Use expression `={{ $('Get Ready Invoices').item.json['Client ID'][0] }}` to get client record ID.  
   - Credentials: Airtable API token.

9. **Add Set Node “Set Company Details”**  
   - Assign static company details:  
     - CompanyName, Address, TaxId, Email, Phone, Logo, Account Number, BIC, Bank Name (use your own values).

10. **Add Custom JS PDF Toolkit Node “Generate Invoice”**  
    - Configure issuer fields with company details from Set Company Details node.  
    - Configure billing fields with notes, tax rate 19%, currency EUR, invoice date and number from “Get Ready Invoices”.  
    - Configure payment fields with bank details.  
    - Pass the aggregated items JSON from “Loop Over Items”.  
    - Configure recipient fields with client info from “Get Clients”.  
    - Credentials: Set up the Custom JS API credentials.

11. **Add Email Send Node “Send Email With Attachment”**  
    - Set subject: “Your Invoice for Last Month”  
    - To: Use dynamic recipient email (replace hardcoded “info@yourcomp.org” with client email ideally).  
    - From: Use invoice email from JSON.  
    - Body: Friendly message text.  
    - Attachments: Use the PDF data from “Generate Invoice”.  
    - Credentials: Configure SMTP credentials.

12. **Add Airtable Node “Update record”**  
    - Operation: Update  
    - Base and Table: Invoice table.  
    - Update “Status” field to “Sent” for the processed invoice ID.  
    - Match by invoice ID from “Get Ready Invoices”.  
    - Credentials: Airtable API token.

13. **Connect nodes according to the flow:**  
    - Manual Trigger → Get Ready Invoices → Loop Over Items → Get Invoice Items → Map Fields → Aggregate → Loop Over Items (for iteration) → Get Clients → Set Company Details → Generate Invoice → Send Email With Attachment → Update record.

14. **Add Sticky Notes for documentation and clarity** as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                                                       |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Public Airtable Example: [Invoice Management With Airtable](https://airtable.com/apphyDa3uYAq0VOMW/shrSe39NZYrqm4gtE) | Airtable base used in this workflow for invoices and clients.                                                                         |
| Screenshot of Workflow: ![InvoiceGeneratorWorkflow.png](https://www.beta.customjs.space/images/integration/n8n/InvoiceGeneratorWorkflow.png) | Visual guide for the workflow design.                                                                                                |
| Invoice generation uses Custom JS PDF Toolkit node available from CustomJS integration in n8n.                  | Custom PDF generation requires valid API credentials from CustomJS services.                                                         |
| Email sending relies on configured SMTP credentials for authentication and delivery.                            | Ensure proper SMTP server settings for successful email dispatch.                                                                     |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.