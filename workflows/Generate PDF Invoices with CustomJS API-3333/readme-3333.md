Generate PDF Invoices with CustomJS API

https://n8nworkflows.xyz/workflows/generate-pdf-invoices-with-customjs-api-3333


# Generate PDF Invoices with CustomJS API

### 1. Workflow Overview

This workflow, titled **Generate PDF Invoices with CustomJS API**, automates the creation of professional PDF invoices from incoming invoice data. It is designed for scenarios where invoice details are received via HTTP requests and need to be converted into styled PDF documents for distribution or storage.

The workflow is logically divided into the following blocks:

- **1.1 Webhook Trigger**: Receives invoice data via an HTTP webhook.
- **1.2 Data Initialization**: Sets or overrides invoice data fields to ensure consistent structure.
- **1.3 Data Preprocessing**: Transforms raw invoice data into HTML fragments suitable for PDF generation.
- **1.4 HTML to PDF Conversion**: Converts the prepared HTML invoice into a PDF document using the CustomJS PDF Toolkit node.
- **1.5 Webhook Response**: Sends the generated PDF back as the HTTP response to the original webhook request.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Trigger

- **Overview:**  
  This node listens for incoming HTTP requests containing invoice data. It acts as the entry point to the workflow, triggering subsequent processing steps.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Type:** Webhook  
  - **Role:** HTTP endpoint to receive invoice data.  
  - **Configuration:**  
    - Path: `526fd864-6f85-4cde-97aa-39b61a3e5b83` (unique webhook URL segment)  
    - Response Mode: `responseNode` (response is sent by a downstream node)  
  - **Input:** HTTP POST or GET request with JSON payload containing invoice fields.  
  - **Output:** Passes received data to the next node (`Set data`).  
  - **Edge Cases / Failures:**  
    - Missing or malformed JSON payload may cause downstream errors.  
    - Unauthorized or unexpected requests are not explicitly handled here.  
  - **Version:** n8n Webhook node v2.

---

#### 1.2 Data Initialization (Set Data)

- **Overview:**  
  This node initializes or overrides invoice data fields to ensure the workflow has consistent and complete invoice information to process.

- **Nodes Involved:**  
  - Set data

- **Node Details:**  
  - **Type:** Set  
  - **Role:** Define static or default invoice data fields.  
  - **Configuration:**  
    - Assigns fields:  
      - `Invoice No`: `"1"`  
      - `Bill To`: multiline string with recipient name and address  
      - `From`: multiline string with sender company and address  
      - `Details`: array of invoice items (each with `description`, `price`, `qty`)  
      - `Email`: contact email string  
  - **Input:** Data from Webhook node (ignored here, replaced by static values).  
  - **Output:** Passes structured invoice data to `Preprocess` node.  
  - **Edge Cases / Failures:**  
    - Since values are hardcoded, no dynamic input validation occurs here.  
    - If the workflow is adapted to receive dynamic data, this node must be updated accordingly.  
  - **Version:** n8n Set node v3.4.

---

#### 1.3 Data Preprocessing

- **Overview:**  
  This code node transforms raw invoice data into HTML-formatted strings suitable for embedding in the PDF. It formats addresses and invoice items into HTML paragraphs and table rows, and calculates the total invoice amount.

- **Nodes Involved:**  
  - Preprocess

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:** Data transformation and HTML generation.  
  - **Configuration:**  
    - Runs once per item (`runOnceForEachItem` mode).  
    - JavaScript code:  
      - Splits `Bill To` and `From` multiline strings by newline, wraps each line in `<p>` tags.  
      - Maps `Details` array into HTML table rows with columns: description, unit price, quantity, total price per item.  
      - Calculates total invoice amount by summing item totals.  
      - Returns an object with keys: `bill_to`, `from`, `details` (HTML string), and `total` (number).  
  - **Input:** JSON object with invoice fields from `Set data`.  
  - **Output:** JSON object with HTML fragments and total amount for PDF generation.  
  - **Edge Cases / Failures:**  
    - Missing or malformed fields (e.g., missing `Details` array) may cause runtime errors.  
    - Assumes `Details` contains valid numeric `price` and `qty`.  
    - No error handling for empty or invalid data.  
  - **Version:** n8n Code node v2.

---

#### 1.4 HTML to PDF Conversion

- **Overview:**  
  Converts the preprocessed HTML invoice into a styled PDF document using the CustomJS PDF Toolkit node.

- **Nodes Involved:**  
  - HTML to PDF

- **Node Details:**  
  - **Type:** `@custom-js/n8n-nodes-pdf-toolkit.html2Pdf` (community node)  
  - **Role:** Generate PDF from HTML input.  
  - **Configuration:**  
    - `htmlInput`: A complete HTML document string with embedded CSS styles and dynamic placeholders.  
    - Uses n8n expressions to inject data from previous nodes:  
      - Invoice number from `Set data` node  
      - `bill_to`, `from`, `details`, and `total` from `Preprocess` node JSON  
      - Email from `Set data` node  
    - Styling includes responsive layout, colors, fonts, table styling, and buttons for a professional invoice look.  
  - **Credentials:** Requires CustomJS API key credential for PDF generation.  
  - **Input:** JSON with HTML fragments and invoice data.  
  - **Output:** Binary PDF file data.  
  - **Edge Cases / Failures:**  
    - API key missing or invalid causes authentication errors.  
    - Large or malformed HTML may cause PDF generation failure or timeouts.  
    - Community node requires self-hosted n8n instance.  
  - **Version:** Node version 1.

---

#### 1.5 Webhook Response

- **Overview:**  
  Sends the generated PDF binary data back as the HTTP response to the original webhook request.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  
  - **Type:** Respond to Webhook  
  - **Role:** Return binary PDF data as HTTP response.  
  - **Configuration:**  
    - Respond with: `binary` data (PDF file).  
  - **Input:** Binary PDF output from `HTML to PDF` node.  
  - **Output:** HTTP response to webhook caller.  
  - **Edge Cases / Failures:**  
    - If input is missing or corrupted, response may fail or be empty.  
    - Timeout or network issues may affect response delivery.  
  - **Version:** n8n Respond to Webhook node v1.1.

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role               | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                          |
|--------------------|----------------------------------|------------------------------|---------------------|----------------------|----------------------------------------------------------------------------------------------------|
| Webhook            | Webhook                          | Entry point, receive invoice data | -                   | Set data             |                                                                                                    |
| Set data           | Set                              | Initialize invoice data       | Webhook             | Preprocess            |                                                                                                    |
| Preprocess         | Code                             | Transform data into HTML      | Set data             | HTML to PDF           |                                                                                                    |
| HTML to PDF        | @custom-js/n8n-nodes-pdf-toolkit.html2Pdf | Convert HTML to PDF           | Preprocess           | Respond to Webhook    | Requires CustomJS API key credential; community node only on self-hosted n8n instances.             |
| Respond to Webhook | Respond to Webhook                | Return PDF as HTTP response   | HTML to PDF          | -                    |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a **Webhook** node.  
   - Set the **Path** to a unique identifier (e.g., `526fd864-6f85-4cde-97aa-39b61a3e5b83`).  
   - Set **Response Mode** to `responseNode`.  
   - This node will receive invoice data via HTTP requests.

2. **Add Set Node (Set data)**  
   - Add a **Set** node connected to the Webhook node.  
   - Configure the following fields with static values (or adapt to dynamic input if desired):  
     - `Invoice No` (string): `"1"`  
     - `Bill To` (string): multiline address, e.g.,  
       ```
       John Doe
       1234 Elm St, Apt 567
       City, Country, 12345
       ```  
     - `From` (string): multiline sender address, e.g.,  
       ```
       ABC Corporation
       789 Business Ave
       City, Country, 67890
       ```  
     - `Details` (array): list of invoice items, each with:  
       - `description` (string)  
       - `price` (number)  
       - `qty` (number)  
     - `Email` (string): contact email, e.g., `"support@mycompany.com"`.

3. **Add Code Node (Preprocess)**  
   - Add a **Code** node connected to the Set node.  
   - Set **Mode** to `runOnceForEachItem`.  
   - Paste the following JavaScript code to transform data:  
     ```javascript
     const input = $input.item.json;
     const bill_to = input['Bill To'].split('\n').map(item => '<p>' + item + '</p>');
     const from = input['From'].split('\n').map(item => '<p>' + item + '</p>');
     const details = input['Details'].map(item => {
       const price = item.price * item.qty;
       return `
       <tr>
         <td>${item.description}</td>
         <td>$${item.price}</td>
         <td>${item.qty}</td>
         <td>$${price}</td>
       </tr>
       `;
     });
     const total = input['Details'].reduce((val, next) => val + next.price * next.qty, 0);
     return {
       bill_to: bill_to.join('\n'),
       from: from.join('\n'),
       details: details.join('\n'),
       total
     };
     ```

4. **Add HTML to PDF Node**  
   - Install the community node `@custom-js/n8n-nodes-pdf-toolkit` if not already installed (requires self-hosted n8n).  
   - Add an **HTML to PDF** node connected to the Preprocess node.  
   - Configure the **HTML Input** parameter with the full HTML template below, embedding expressions to inject data dynamically:  
     ```html
     <!DOCTYPE html>
     <html lang="en">
     <head>
         <meta charset="UTF-8" />
         <meta name="viewport" content="width=device-width, initial-scale=1.0" />
         <title>Invoice</title>
         <style>
           /* Include all CSS styles as per the original template */
           /* ... (copy CSS from the original node) ... */
         </style>
     </head>
     <body>
       <div class="invoice-wrapper">
         <div class="header">
           <h1>Invoice</h1>
           <p>Invoice #{{ $('Set data').item.json['Invoice No'] }}</p>
         </div>
         <div class="invoice-details">
           <div>
             <h3>Billed To:</h3>
             {{ $json.bill_to }}
           </div>
           <div>
             <h3>From:</h3>
             {{ $json.from }}
             <p>Email: {{ $('Set data').item.json.Email }}</p>
           </div>
         </div>
         <table class="table">
           <thead>
             <tr>
               <th>Description</th>
               <th>Unit Price</th>
               <th>Quantity</th>
               <th>Total</th>
             </tr>
           </thead>
           <tbody>
             {{ $json.details }}
             <tr class="total">
               <td colspan="3">Total Amount</td>
               <td>${{ $json.total }}</td>
             </tr>
           </tbody>
         </table>
         <div class="footer">
           <p>Thank you for doing business with us!</p>
           <p>If you have any questions regarding this invoice, please contact us at <a href="mailto:contact@abccorp.com">{{ $('Set data').item.json.Email }}</a>.</p>
           <a href="mailto:{{ $('Set data').item.json.Email }}" class="btn">Contact Us</a>
         </div>
       </div>
     </body>
     </html>
     ```  
   - Assign the **CustomJS API** credential with your API key.

5. **Add Respond to Webhook Node**  
   - Add a **Respond to Webhook** node connected to the HTML to PDF node.  
   - Set **Respond With** to `binary` to send the PDF file as the HTTP response.

6. **Connect Nodes**  
   - Connect nodes in this order:  
     `Webhook` → `Set data` → `Preprocess` → `HTML to PDF` → `Respond to Webhook`.

7. **Credential Setup**  
   - Create a credential in n8n for the CustomJS API key:  
     - Go to Credentials → New Credential → CustomJS API.  
     - Enter your API key obtained from https://www.customjs.space.  
   - Assign this credential to the HTML to PDF node.

8. **Testing**  
   - Trigger the webhook URL with JSON payload matching the invoice data structure or rely on the static data in the Set node.  
   - The response should be a PDF file with the formatted invoice.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Community nodes like `@custom-js/n8n-nodes-pdf-toolkit` require self-hosted n8n instances.          | https://docs.n8n.io/integrations/community-nodes/ |
| Obtain CustomJS API key by signing up at [CustomJS](https://www.customjs.space).                     | https://www.customjs.space                        |
| Example invoice data JSON provided in the workflow description for testing and customization.       | See "Example Invoice Data" section above         |
| The HTML template uses inline CSS for styling the invoice, including responsive design and branding. | Embedded in the HTML to PDF node configuration   |
| For more information on webhook usage in n8n, see official docs: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/ | n8n Webhook documentation                         |

---

This document fully describes the **Generate PDF Invoices with CustomJS API** workflow, enabling advanced users and automation agents to understand, reproduce, and modify the workflow effectively.