Client Billing Detail Collection & Invoice Generation with Gmail and QuickBooks

https://n8nworkflows.xyz/workflows/client-billing-detail-collection---invoice-generation-with-gmail-and-quickbooks-6505


# Client Billing Detail Collection & Invoice Generation with Gmail and QuickBooks

### 1. Workflow Overview

This workflow automates the process of collecting client billing details and generating invoices using Gmail and QuickBooks Online (QBO). It is designed for freelancers, small businesses, or teams who want to streamline invoicing by automating client communication, billing data collection, customer management in QBO, invoice creation, and invoice delivery.

The workflow logically divides into these functional blocks:

- **1.1 Input Reception**  
  Captures initial client and invoice details via a hosted form to start the process.

- **1.2 Client Billing Info Request and Collection**  
  Sends an email to the client requesting detailed billing information via a form, then collects the client’s billing data when submitted.

- **1.3 Client Management in QuickBooks**  
  Checks if the client exists in QuickBooks; if not, creates a new customer record.

- **1.4 Product Lookup**  
  Retrieves the selected product details from QuickBooks based on the client's choice.

- **1.5 Invoice Creation and Delivery**  
  Generates a draft invoice in QuickBooks using collected data, then sends the invoice to the client via QuickBooks native email functionality.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by gathering essential client details, invoice amount, product, description, and due date through a user-submitted form.

- **Nodes Involved:**  
  - Enter Client Details (formTrigger)

- **Node Details:**  

  - **Enter Client Details**  
    - *Type & Role:* Form Trigger Node; entry point for collecting client and invoice data.  
    - *Configuration:*  
      - Form fields include client's first name, email, product (dropdown with "Item A" and "Misc"), description, amount (number), and invoice due date.  
      - Form title and description clarify usage.  
      - Attribution appended automatically.  
    - *Expressions:* Form values are referenced downstream using `$('Enter Client Details').item.json`.  
    - *Input/Output:* No incoming connections; outputs client data to "Ask Client for Billing Info."  
    - *Edge Cases:*  
      - Missing required fields prevent submission.  
      - Dropdown product names must exactly match QuickBooks items to avoid lookup failures.  
    - *Sticky Notes:*  
      - Advises saving production URL for external use.  
      - Notes that filling this form starts the workflow.

#### 2.2 Client Billing Info Request and Collection

- **Overview:**  
  Sends a customized email to the client requesting detailed billing information, then collects the response via a built-in email form.

- **Nodes Involved:**  
  - Ask Client for Billing Info (Gmail)  

- **Node Details:**  

  - **Ask Client for Billing Info**  
    - *Type & Role:* Gmail Node with "sendAndWait" operation; sends an email with a form to the client and waits for their response.  
    - *Configuration:*  
      - Email sent to the client’s email collected from the initial form.  
      - Message includes invoice description and amount, requesting billing info.  
      - Custom form fields requested: first name, last name, company name, street address, city, state/province, postal code, phone number.  
      - Response type set to custom form, allowing structured data collection.  
    - *Expressions:* Dynamic fields like client first name, description, and amount pulled from prior form data.  
    - *Input/Output:* Input from "Enter Client Details"; outputs billing info data to "Add Client to QBO."  
    - *Edge Cases:*  
      - Email deliverability issues (spam filters, invalid addresses) may cause failures.  
      - Client may not respond, causing workflow to wait indefinitely unless manually stopped or timed out externally.  
      - Missing or incorrect email credentials cause auth errors.  
    - *Sticky Notes:*  
      - Explains this step sends an official Gmail email to request billing info.

#### 2.3 Client Management in QuickBooks

- **Overview:**  
  This block verifies if the client already exists in QuickBooks by searching for a customer with the same display name; if not found, it creates a new client record with the collected billing details.

- **Nodes Involved:**  
  - Add Client to QBO (QuickBooks)  
  - Find Existing Customer (QuickBooks)  

- **Node Details:**

  - **Add Client to QBO**  
    - *Type & Role:* QuickBooks Node; creates a new customer record in QBO.  
    - *Configuration:*  
      - Customer display name is set to company name if provided; otherwise, concatenation of first and last name.  
      - Billing address fields populated from client billing form submission.  
      - Contact details (phone, email) set from form data.  
      - Uses OAuth2 credentials for QuickBooks.  
    - *Expressions:* Uses `$json.data` fields from client billing response.  
    - *Input/Output:* Input from "Ask Client for Billing Info"; output connects to "Find Existing Customer."  
    - *Error Handling:* Configured to continue workflow even if creation fails (onError: continueRegularOutput), allowing subsequent steps to proceed.  
    - *Edge Cases:*  
      - Duplicate customer creation if search logic fails or is too narrow.  
      - API rate limits or auth expiration can cause errors.  

  - **Find Existing Customer**  
    - *Type & Role:* QuickBooks Node; searches for existing customer by display name.  
    - *Configuration:*  
      - Query filters customers where DisplayName equals the company name or full name from the initial client info form.  
      - Limits results to 1 (first match).  
    - *Expressions:* Uses billing info data from "Ask Client for Billing Info" node JSON.  
    - *Input/Output:* Input from "Add Client to QBO"; output to "Get The Selected Product."  
    - *Edge Cases:*  
      - Customer name matching might fail due to inconsistent naming conventions.  
      - No customer found results in empty output, triggering client creation handling downstream.

    - *Sticky Notes:*  
      - "Add Client to QBO" and "Find Existing Customer" have notes describing their respective roles.

#### 2.4 Product Lookup

- **Overview:**  
  Retrieves product details from QuickBooks matching the product selected by the user in the initial form.

- **Nodes Involved:**  
  - Get The Selected Product (QuickBooks)  

- **Node Details:**

  - **Get The Selected Product**  
    - *Type & Role:* QuickBooks Node; fetches item details based on product name.  
    - *Configuration:*  
      - Query filters items where the name equals the product selected in the initial form.  
      - Limits results to 1.  
    - *Expressions:* Uses product selection from `'Enter Client Details'` node.  
    - *Input/Output:* Input from "Find Existing Customer"; output to "Create A New Invoice."  
    - *Edge Cases:*  
      - Product name mismatch leads to no results, causing invoice creation to fail due to missing item ID.  
      - API errors or permissions issues with QuickBooks.  
    - *Sticky Notes:*  
      - Explains this node fetches the product selected from QuickBooks.

#### 2.5 Invoice Creation and Delivery

- **Overview:**  
  Creates an invoice in QuickBooks with the collected client and product data, then sends the invoice email to the client.

- **Nodes Involved:**  
  - Create A New Invoice (QuickBooks)  
  - Send the Invoice (QuickBooks)  

- **Node Details:**  

  - **Create A New Invoice**  
    - *Type & Role:* QuickBooks Node; creates a draft invoice with line item details.  
    - *Configuration:*  
      - Invoice line includes amount, item ID from product lookup, description, and tax code (hardcoded as "5").  
      - Customer reference uses existing customer ID if found, otherwise uses newly created customer ID.  
      - Due date and billing email set from initial form data.  
    - *Expressions:* Combines data from multiple prior nodes for line item and customer details.  
    - *Input/Output:* Input from "Get The Selected Product"; output to "Send the Invoice."  
    - *Edge Cases:*  
      - Missing or invalid customer or product IDs cause creation failure.  
      - Tax code hardcoding may require adjustment based on region or product.  
      - API limits or auth errors possible.  
    - *Sticky Notes:*  
      - Notes that this node builds a draft invoice using client and product info.

  - **Send the Invoice**  
    - *Type & Role:* QuickBooks Node; sends the created invoice to the client email.  
    - *Configuration:*  
      - Uses invoice ID from created invoice.  
      - Sends email to client email from the initial form.  
    - *Input/Output:* Input from "Create A New Invoice"; no outputs (end of workflow).  
    - *Edge Cases:*  
      - Email sending failure due to invalid emails or QuickBooks email service issues.  
      - Invoice ID missing or invalid causes failure to send.  
    - *Sticky Notes:*  
      - Explains this node sends the finished invoice to the client.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                             | Input Node(s)              | Output Node(s)           | Sticky Note                                                          |
|-------------------------|---------------------|--------------------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------|
| Sticky Note             | Sticky Note         | Workflow introduction and overview         |                            |                          | Detailed workflow description, use cases, setup, and customization   |
| Sticky Note1            | Sticky Note         | Notes on Enter Client Details node         |                            |                          | Advises saving URL and form usage                                    |
| Sticky Note2            | Sticky Note         | Notes on Ask Client for Billing Info node  |                            |                          | Explains this node sends an official Gmail email                    |
| Sticky Note5            | Sticky Note         | Notes on Add Client to QBO node             |                            |                          | Explains purpose of adding client to QuickBooks                      |
| Sticky Note6            | Sticky Note         | Notes on Find Existing Customer node        |                            |                          | Explains purpose of customer search in QuickBooks                    |
| Sticky Note7            | Sticky Note         | Notes on Create A New Invoice node          |                            |                          | Describes invoice building process                                  |
| Sticky Note8            | Sticky Note         | Notes on Send the Invoice node              |                            |                          | Explains sending the invoice to client                              |
| Sticky Note9            | Sticky Note         | Notes on Get The Selected Product node      |                            |                          | Explains product lookup from QuickBooks                             |
| Enter Client Details     | Form Trigger        | Collects initial client and invoice details|                            | Ask Client for Billing Info| Advises saving production URL                                        |
| Ask Client for Billing Info | Gmail            | Sends billing info request email and collects response | Enter Client Details         | Add Client to QBO          | Sends official Gmail email requesting billing info                 |
| Add Client to QBO        | QuickBooks          | Creates new customer record if needed      | Ask Client for Billing Info | Find Existing Customer     | Adds client to QuickBooks if not found                              |
| Find Existing Customer   | QuickBooks          | Searches for existing customer              | Add Client to QBO           | Get The Selected Product   | Searches QuickBooks for customer by display name                    |
| Get The Selected Product | QuickBooks          | Retrieves selected product details          | Find Existing Customer      | Create A New Invoice       | Fetches product information from QuickBooks                        |
| Create A New Invoice     | QuickBooks          | Creates invoice draft                        | Get The Selected Product    | Send the Invoice           | Builds invoice using client and product data                       |
| Send the Invoice         | QuickBooks          | Sends invoice email to client                | Create A New Invoice        |                          | Sends the finished invoice to the client                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Enter Client Details" Node**  
   - Node Type: Form Trigger  
   - Configure form title: "Enter Client Details"  
   - Add required fields:  
     - First Name (text, required)  
     - Email (text, required)  
     - Product (dropdown, required) with options: "Item A", "Misc"  
     - Description (textarea, required)  
     - Amount (number, required)  
     - Invoice Due Date (date, required)  
   - Enable "Append Attribution" option  
   - This node serves as the workflow entry point.

2. **Create the "Ask Client for Billing Info" Node**  
   - Node Type: Gmail  
   - Operation: "sendAndWait" (send email and wait for response)  
   - Credentials: Connect your Gmail OAuth2 credential  
   - Send To: Set to `={{ $json['What is the client\'s email?'] }}` from the form trigger  
   - Subject: "Billing Details Required"  
   - Message: Use expression to include client's first name, invoice description, and amount dynamically.  
   - Form fields in email: First Name, Last Name, Company Name (optional), Street Address, City, State / Province, Zip / Postal Code, Phone Number  
   - Response type: Custom Form  
   - Connect this node's input to "Enter Client Details" output.

3. **Create the "Add Client to QBO" Node**  
   - Node Type: QuickBooks  
   - Operation: Create customer  
   - Credentials: Connect your QuickBooks Online OAuth2 credential  
   - Display Name: Set dynamically to company name if present, else first + last name  
   - Additional fields:  
     - Billing Address: City, Street Address, Postal Code, State/Province from billing info response  
     - Contact: GivenName, FamilyName, CompanyName, Phone Number, Primary Email Address from billing info  
   - Configure "On Error" to "continueRegularOutput" to allow workflow continuation if customer exists  
   - Connect input from "Ask Client for Billing Info" output.

4. **Create the "Find Existing Customer" Node**  
   - Node Type: QuickBooks  
   - Operation: Get All customers with filter  
   - Filter Query: Search where DisplayName equals company name or full name from billing info  
   - Limit: 1 result  
   - Credentials: Same QuickBooks credential as before  
   - Connect input to "Add Client to QBO" output.

5. **Create the "Get The Selected Product" Node**  
   - Node Type: QuickBooks  
   - Operation: Get All items with filter  
   - Filter Query: Where name equals the product selected in the initial form trigger  
   - Limit: 1 result  
   - Connect input to "Find Existing Customer" output.

6. **Create the "Create A New Invoice" Node**  
   - Node Type: QuickBooks  
   - Operation: Create invoice  
   - Line item configuration:  
     - Amount: From initial form amount field  
     - Item ID: From "Get The Selected Product" node output  
     - Description: From initial form description field  
     - TaxCodeRef: Set tax code (e.g., "5") — adjust based on your tax setup  
   - CustomerRef: Use ID from "Find Existing Customer" if exists; otherwise, from "Add Client to QBO"  
   - Additional fields: Invoice due date and billing email from initial form  
   - Connect input to "Get The Selected Product" output.

7. **Create the "Send the Invoice" Node**  
   - Node Type: QuickBooks  
   - Operation: Send invoice  
   - Invoice ID: From "Create A New Invoice" output  
   - Email: Client email from initial form  
   - Connect input to "Create A New Invoice" output.

8. **Connect Nodes in Order:**  
   - Enter Client Details → Ask Client for Billing Info → Add Client to QBO → Find Existing Customer → Get The Selected Product → Create A New Invoice → Send the Invoice

9. **Credential Setup:**  
   - Create and authenticate Gmail OAuth2 credential for sending emails.  
   - Create and authenticate QuickBooks Online OAuth2 credential with permissions to manage customers, items, and invoices.

10. **Final Checks and Defaults:**  
    - Confirm that QuickBooks product names match exactly with dropdown options to avoid product lookup failures.  
    - Ensure tax code in invoice creation is correct for your region.  
    - Customize email message in "Ask Client for Billing Info" node to match brand voice.  
    - Save and publish the form URL from "Enter Client Details" node to trigger workflow externally.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| This workflow removes friction from billing by automating client data collection and invoice generation with minimal manual intervention. It is ideal for freelancers and small businesses wanting to reduce errors and save time.                                                                                                                                                     | Workflow purpose and benefits           |
| When using the form trigger node, save and reuse the published URL to start the workflow outside of n8n’s interface.                                                                                                                                                                                                                                                                | Enter Client Details node note          |
| Ensure that product names in the form dropdown exactly match QuickBooks item names to avoid failures during product lookup.                                                                                                                                                                                                                                                        | Product matching requirement            |
| Customize the email message in the Gmail node to reflect your brand voice and provide clear instructions to clients.                                                                                                                                                                                                                                                               | Email customization advice              |
| Tax codes in the invoice creation node are hardcoded; adjust these to your local tax regulations or QuickBooks setup.                                                                                                                                                                                                                                                               | Tax code configuration note             |
| OAuth2 credentials for Gmail and QuickBooks Online must be created and authorized with appropriate scopes before using this workflow.                                                                                                                                                                                                                                              | Credential prerequisites                |
| Potential extensions include adding reminder emails for incomplete client billing forms or logging invoice status in Google Sheets or Airtable.                                                                                                                                                                                                                                    | Customization suggestions               |
| Official n8n documentation for QuickBooks integration and Gmail OAuth2 setup can provide additional configuration help: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.quickbooks/ and https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                                                                                             | Official documentation links            |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.