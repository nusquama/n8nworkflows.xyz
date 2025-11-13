Automate E-commerce Return Guides with Email Verification, PDF/Image Generation & QR Codes

https://n8nworkflows.xyz/workflows/automate-e-commerce-return-guides-with-email-verification--pdf-image-generation---qr-codes-8942


# Automate E-commerce Return Guides with Email Verification, PDF/Image Generation & QR Codes

### 1. Workflow Overview

This workflow automates the generation and delivery of verified product return guides for an e-commerce platform. It targets customer service teams and e-commerce operations aiming to streamline return processes while preventing fraud through email verification. The workflow:

- Receives return requests via a webhook,
- Validates customer emails to prevent fake or disposable addresses,
- Enriches data with return instructions, deadlines, QR codes, and tracking URLs,
- Generates a styled HTML return guide,
- Converts the guide into both PDF and image formats,
- Emails the approved return guide with attachments and instructions to the verified customer.

Logical blocks grouped by function and dependencies:

- **1.1 Input Reception & Email Verification:** Accepting return requests and validating customer emails.
- **1.2 Data Preparation:** Merging verified email results with original data and adding calculated fields.
- **1.3 HTML Guide Generation:** Creating a professional, styled HTML return guide with embedded data.
- **1.4 PDF and Image Conversion:** Simultaneously converting HTML guides to PDF and image formats.
- **1.5 Email Delivery:** Sending the return guide via email with links and instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Email Verification

**Overview:**  
Receives incoming product return requests via HTTP POST, then verifies the provided customer email to prevent fraud and fake returns.

**Nodes Involved:**  
- Webhook  
- Verifi Email  
- Switch  
- Stop and Error  

**Node Details:**

- **Webhook**  
  - Type: Webhook (HTTP Trigger)  
  - Configuration: Listens on POST `/return-guide` path; expects JSON payload with keys like customer_name, customer_email, order_id, etc.  
  - Inputs: External HTTP POST calls  
  - Outputs: JSON data forwarded to email verification  
  - Edge Cases: Missing or malformed JSON, incorrect HTTP method  
  - Sticky Note: Explains expected JSON schema and test URL.  

- **Verifi Email**  
  - Type: Email Verification Node (third-party API)  
  - Configuration: Verifies email passed from webhook (`customer_email` field) using VerifiEmail API credentials  
  - Inputs: Email address from webhook JSON  
  - Outputs: Verification result with a boolean `valid` field  
  - Edge Cases: API auth failure, rate limiting, network issues  
  - Credential: VerifiEmail API key required  
  - Sticky Note: Describes purpose (fraud prevention) and credential requirements  

- **Switch**  
  - Type: Conditional Router  
  - Configuration: Checks if `valid` field from Verifi Email node is true or false  
  - Inputs: Verification result JSON  
  - Outputs:  
    - True path: proceeds to data merging  
    - False path: triggers error stop node  
  - Edge Cases: Missing `valid` field or unexpected data types  

- **Stop and Error**  
  - Type: Workflow Stop with Error  
  - Configuration: Stops execution and returns error message "Email validation failed"  
  - Inputs: Invalid email path from Switch  
  - Outputs: None (workflow terminates)  

---

#### 1.2 Data Preparation

**Overview:**  
Combines the verified email data with the original webhook data, enriches it with calculated fields like QR code URL, return deadlines, and tracking URLs, then cleans duplicates.

**Nodes Involved:**  
- Merge  
- Edit Fields (Set)  
- Remove Duplicates  

**Node Details:**

- **Merge**  
  - Type: Data Merger  
  - Configuration: Combines outputs of Webhook and Verifi Email nodes (email verification + original request) into single JSON object  
  - Inputs: Two inputs (Webhook, Verifi Email)  
  - Outputs: Merged JSON for further processing  
  - Edge Cases: Missing data from either input could cause incomplete merges  

- **Edit Fields (Set)**  
  - Type: Set Node (Field Assignment)  
  - Configuration: Adds/calculates multiple fields:  
    - `instructions`: Static multiline return instructions string  
    - `qr_url`: Generates QR code URL embedding the order ID via external QR code API  
    - `return_deadline`: Sets return deadline as 7 days from current date  
    - `tracking_url`: Constructs tracking URL using order ID  
    - Copies customer_name, customer_email, order_id, return_reason, product_name, purchase_date from merged input  
  - Inputs: Merged JSON  
  - Outputs: Enriched data JSON  
  - Edge Cases: Missing or malformed dates, order_id, or customer info could affect URL generation  

- **Remove Duplicates**  
  - Type: Deduplication Node  
  - Configuration: Cleans potential duplicate records in merged/enriched data  
  - Inputs: Enriched data JSON  
  - Outputs: Distinct data items for next steps  
  - Edge Cases: Incorrect deduplication criteria may remove valid items  

---

#### 1.3 HTML Guide Generation

**Overview:**  
Generates a fully styled, professional HTML return authorization guide using the cleaned and enriched data. The HTML includes QR codes, customer details, instructions, and branding.

**Nodes Involved:**  
- Code in JavaScript  

**Node Details:**

- **Code in JavaScript**  
  - Type: Code Node  
  - Configuration:  
    - Cleans field names with leading spaces  
    - Constructs a complete HTML document string embedding all relevant data (order ID, customer name/email, product info, return instructions, QR code, deadline, tracking URL)  
    - Applies modern CSS styling for readability and branding  
    - Includes placeholders for product images and support contact info  
    - Outputs JSON with `html_content`, `customer_name`, `customer_email`, `order_id`, and `file_name` for downstream use  
  - Inputs: Deduplicated enriched data  
  - Outputs: HTML content JSON  
  - Edge Cases: Missing data fields could cause incomplete HTML or broken links/images  
  - Notes: Uses template literals and inline CSS, no external dependencies  

---

#### 1.4 PDF and Image Conversion

**Overview:**  
Converts the generated HTML return guide into both PDF (for official/print use) and PNG image (for email preview) formats in parallel batches.

**Nodes Involved:**  
- Loop Over Items (Split in Batches)  
- HTML to PDF  
- HTML/CSS to Image  
- Merge1  

**Node Details:**

- **Loop Over Items**  
  - Type: Split in Batches  
  - Configuration: Processes each item individually (batch size 1)  
  - Inputs: HTML content JSON array from Code in JavaScript node  
  - Outputs: Single items to parallel conversion nodes  
  - Edge Cases: Large batches could cause performance issues if not reset  

- **HTML to PDF**  
  - Type: HTML to PDF Conversion (third-party API)  
  - Configuration: Converts HTML content string into PDF document  
  - Inputs: `html_content` string from loop item  
  - Outputs: PDF file metadata including hosted URL  
  - Credential: HtmlCssToPdf API key required  
  - Edge Cases: API rate limits, malformed HTML causing conversion errors  

- **HTML/CSS to Image**  
  - Type: HTML to Image Conversion (third-party API)  
  - Configuration: Converts same HTML into PNG image for preview  
  - Inputs: `html_content` string from loop item  
  - Outputs: Image file metadata including hosted URL  
  - Credential: HtmlCssToImage API key required  
  - Edge Cases: Same as PDF conversion  

- **Merge1**  
  - Type: Data Merger (Combine Mode)  
  - Configuration: Combines results from both PDF and Image conversion nodes by matching on `success` status  
  - Inputs: Outputs of HTML to PDF and HTML/CSS to Image nodes  
  - Outputs: Merged data including URLs for both PDF and image files  
  - Edge Cases: Unmatched or missing conversion results may disrupt email sending  

---

#### 1.5 Email Delivery

**Overview:**  
Sends a personalized email to the verified customer with return guide download links, instructions, and deadlines.

**Nodes Involved:**  
- Send a message (Gmail)  

**Node Details:**

- **Send a message**  
  - Type: Gmail Node (OAuth2 Auth)  
  - Configuration:  
    - Recipient: Verified customer email from Loop Over Items node  
    - Subject: Includes order ID  
    - Message Body: Plain text email with return guide approval notice, PDF and image download links, step-by-step next steps, deadline warning, and support contact info  
  - Inputs: Merged data containing URLs and customer info  
  - Credential: Gmail OAuth2 credentials required  
  - Edge Cases: Email sending failures due to auth issues, invalid email addresses, or rate limits  
  - Notes: Email text dynamically constructed with expressions referencing various nodes  

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                   | Input Node(s)                  | Output Node(s)            | Sticky Note                                                                                                                        |
|--------------------|-------------------------------|---------------------------------|-------------------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Webhook            | Webhook                       | Receive return request via HTTP | External HTTP POST             | Verifi Email, Merge        | Describes webhook path, method, expected JSON, and test URL                                                                       |
| Verifi Email       | Email Verification            | Validate customer email          | Webhook                       | Switch                    | Explains fraud prevention purpose, credential needs, free tier limit                                                               |
| Switch             | Conditional Router            | Branch on email validity         | Verifi Email                  | Merge (valid), Stop and Error (invalid) |                                                                                                                                  |
| Stop and Error     | Workflow Stop/Error           | Stop workflow if email invalid   | Switch (invalid path)          | None                      |                                                                                                                                  |
| Merge              | Data Merger                  | Combine webhook & verification   | Webhook, Verifi Email         | Edit Fields               | Data preparation steps described                                                                                                  |
| Edit Fields (Set)  | Data Enrichment               | Add calculated fields            | Merge                        | Remove Duplicates          | Details data enrichment: QR code URL, deadline, tracking URL, instructions                                                        |
| Remove Duplicates   | Deduplication                | Clean duplicate data             | Edit Fields                  | Code in JavaScript         |                                                                                                                                  |
| Code in JavaScript | Code Node                    | Generate HTML return guide       | Remove Duplicates            | Loop Over Items            | Describes HTML template generation with styling, data embedding, file naming                                                      |
| Loop Over Items     | Split in Batches             | Process each item in batch       | Code in JavaScript           | HTML to PDF, HTML/CSS to Image | Explains parallel conversion setup                                                                                               |
| HTML to PDF        | HTML to PDF Conversion       | Convert HTML to PDF format       | Loop Over Items              | Merge1                    | Describes PDF conversion via API                                                                                                 |
| HTML/CSS to Image  | HTML to Image Conversion      | Convert HTML to PNG image        | Loop Over Items              | Merge1                    | Describes image conversion via API                                                                                               |
| Merge1             | Data Merger (Combine mode)    | Combine PDF and image outputs    | HTML to PDF, HTML/CSS to Image | Send a message             |                                                                                                                                  |
| Send a message      | Gmail Email Node             | Send return guide email          | Merge1                      | None                      | Details email content, dynamic fields, instructions, and credentials requirement                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `/return-guide`  
   - HTTP Method: POST  
   - No authentication required  
   - Expected JSON input fields: `customer_name`, `customer_email`, `order_id`, `return_reason`, `product_name`, `purchase_date`  

2. **Add Verifi Email Node**  
   - Type: Verifi Email (requires VerifiEmail API credentials)  
   - Email parameter: `={{ $json.body.customer_email }}` (extract from webhook JSON)  
   - Connect Webhook main output to Verifi Email input  

3. **Add Switch Node**  
   - Configure two cases based on `{{$json.valid}}`:  
     - Case 1: Equals `true` → valid email path  
     - Case 2: Equals `false` → invalid email path  
   - Connect Verifi Email output to Switch input  

4. **Add Stop and Error Node**  
   - Error message: "Email validation failed"  
   - Connect Switch invalid path to this node  

5. **Add Merge Node**  
   - Connect Webhook output to Merge input 1 (index 1)  
   - Connect Switch valid path output to Merge input 0 (index 0)  
   - This merges webhook original data and verification result  

6. **Add Set Node (Edit Fields)**  
   - Add fields:  
     - `instructions`: Static multiline string with return steps  
     - `qr_url`: `"https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=" + order_id`  
     - `return_deadline`: Current date + 7 days, formatted as "yyyy-MM-dd"  
     - `tracking_url`: `"https://track.example.com/" + order_id`  
     - Copy customer_name, customer_email, order_id, return_reason, product_name, purchase_date from merged data  
   - Connect Merge output to Set node input  

7. **Add Remove Duplicates Node**  
   - Default settings for deduplication  
   - Connect Set node output to Remove Duplicates input  

8. **Add Code Node (JavaScript)**  
   - Write JS code to:  
     - Clean unwanted spaces in field names  
     - Generate a full HTML document string embedding all relevant fields with CSS styling  
     - Output JSON with `html_content`, `customer_name`, `customer_email`, `order_id`, `file_name` = `return_guide_{order_id}`  
   - Connect Remove Duplicates output to Code node input  

9. **Add Split In Batches Node (Loop Over Items)**  
   - Batch size: 1  
   - Connect Code node output to Split in Batches input  

10. **Add HTML to PDF Node**  
    - Use HtmlCssToPdf API credentials  
    - HTML content parameter: `={{ $json.html_content }}` from loop item  
    - Connect Split in Batches first output to this node  

11. **Add HTML/CSS to Image Node**  
    - Use HtmlCssToImage API credentials  
    - HTML content parameter: Same as PDF node  
    - Connect Split in Batches second output to this node  

12. **Add Merge Node (Merge1)**  
    - Mode: Combine  
    - Connect HTML to PDF and HTML/CSS to Image outputs to Merge1 inputs  

13. **Add Gmail Node (Send a message)**  
    - Use Gmail OAuth2 credentials  
    - To: `={{ $('Loop Over Items').item.json.customer_email }}`  
    - Subject: `Return Guide for Order {{ $('Loop Over Items').item.json.order_id }} - Action Required`  
    - Message body: Plain text with dynamic insertion of customer name, order ID, PDF URL, image URL, instructions, deadlines, and support contact  
    - Connect Merge1 output to Gmail node input  

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Webhook example JSON payload and test URL specified in sticky note for testing and integration                   | Webhook node sticky note                                       |
| Email verification prevents fraud from disposable, malformed, or non-existent emails                             | Email Verification sticky note                                 |
| Data preparation includes QR code generation using free API (https://api.qrserver.com)                           | Data Preparation sticky note                                   |
| HTML generation uses embedded CSS styling for professional look                                                  | HTML Template Generator sticky note                            |
| Parallel HTML to PDF and image conversion via paid APIs with free tiers                                          | Parallel Conversion sticky note                                |
| Automated email includes download links, instructions, deadlines, and contact info                               | Automated Email Delivery sticky note                           |
| Credential setup required: VerifiEmail API, Gmail OAuth2, HtmlCssToPdf API, and HtmlCssToImage API                | Credential Setup sticky note                                   |
| Customization options include email template, styling, QR code size, deadlines, branding, and multi-language support | Customization Options sticky note                              |

---

**Disclaimer:** The provided text and workflow are fully compliant with current content policies. It handles only legal and public data accordingly.