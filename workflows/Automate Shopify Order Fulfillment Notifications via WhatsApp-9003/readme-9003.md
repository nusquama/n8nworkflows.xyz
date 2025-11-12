Automate Shopify Order Fulfillment Notifications via WhatsApp

https://n8nworkflows.xyz/workflows/automate-shopify-order-fulfillment-notifications-via-whatsapp-9003


# Automate Shopify Order Fulfillment Notifications via WhatsApp

### 1. Workflow Overview

This workflow automates sending WhatsApp notifications to customers when their Shopify orders are fulfilled. It listens for fulfillment creation events from Shopify, retrieves detailed order and customer information, cleans and validates customer phone numbers for WhatsApp compatibility via the Rapiwa API, sends personalized WhatsApp messages with tracking information, and logs all interactions in Google Sheets with verification status. The workflow includes batching and waiting mechanisms to manage API rate limits and ensure reliability.

The workflow logic is grouped into the following blocks:

- **1.1 Shopify Fulfillment Event Reception:** Captures Shopify fulfillment creation events.
- **1.2 Data Extraction and Formatting:** Processes raw webhook data to extract relevant fulfillment and customer details.
- **1.3 Customer Data Retrieval and Cleaning:** Fetches detailed customer data using Shopify API and cleans/formats phone numbers.
- **1.4 Batch Processing:** Splits data for sequential handling.
- **1.5 WhatsApp Number Verification:** Validates if customer phone numbers are registered on WhatsApp via Rapiwa API.
- **1.6 Conditional Messaging & Logging:** Sends WhatsApp messages to verified numbers, logs verified and unverified contacts separately in Google Sheets.
- **1.7 Throttling and Looping:** Waits between processing batches to avoid rate limiting.

---

### 2. Block-by-Block Analysis

#### 1.1 Shopify Fulfillment Event Reception

- **Overview:**  
  This block triggers the workflow whenever a new fulfillment is created in Shopify.

- **Nodes Involved:**  
  - Shopify Trigger

- **Node Details:**

  - **Shopify Trigger**  
    - *Type:* Shopify Trigger (Webhook)  
    - *Role:* Listens for `fulfillments/create` events from Shopify store.  
    - *Configuration:*  
      - Topic: `fulfillments/create`  
      - Auth: Shopify Access Token (configured with OAuth token credentials)  
      - Webhook ID present for event reception  
    - *Input:* External webhook call triggered by Shopify.  
    - *Output:* Raw fulfillment payload containing fulfillment and order data.  
    - *Edge Cases:*  
      - Failed webhook registration or revoked permissions.  
      - Shopify API version updates (using 2025-07).  
      - Network or authentication errors.  
    - *Sub-workflow:* None

#### 1.2 Data Extraction and Formatting

- **Overview:**  
  Extracts and formats key data from the webhook payload for easier downstream processing.

- **Nodes Involved:**  
  - Format Webhook Response Data

- **Node Details:**

  - **Format Webhook Response Data**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Parses raw webhook data array, extracts fulfillment and order details, customer info, product titles, tracking info, and shop domain.  
    - *Configuration:* Custom JS code mapping input JSON to simplified object structure.  
    - *Key Expressions:*  
      Accesses `body.id`, `body.order_id`, `body.tracking_number`, `body.line_items`, `body.destination`, and webhook headers.  
    - *Input:* Raw webhook JSON from Shopify Trigger.  
    - *Output:* Array of objects with selected fields (customer_id, fulfillment_id, order_id, status, tracking info, product title, customer name, country, shop domain).  
    - *Edge Cases:*  
      - Missing or malformed webhook data fields.  
      - Empty or undefined arrays like `line_items`.  
    - *Sub-workflow:* None

#### 1.3 Customer Data Retrieval and Cleaning

- **Overview:**  
  Retrieves full customer data per order from Shopify API and cleans phone numbers, preparing data for WhatsApp validation.

- **Nodes Involved:**  
  - get all customer data  
  - Loop Over Items  
  - Clean Number

- **Node Details:**

  - **get all customer data**  
    - *Type:* HTTP Request  
    - *Role:* Uses `order_id` to fetch full Shopify order data including customer info.  
    - *Configuration:*  
      - Method: GET  
      - URL: `https://your_domain.myshopify.com/admin/api/2025-07/orders/{{ $json.order_id }}.json` (dynamic order_id)  
      - Header: `X-Shopify-Access-Token` with Shopify API access token  
    - *Input:* Formatted data from previous node with order_id.  
    - *Output:* Full order JSON including customer details.  
    - *Edge Cases:*  
      - API rate limits or authentication errors.  
      - Non-existent order IDs or orders without customers.  
      - Network timeouts.  
    - *Sub-workflow:* None

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes each customer/order item sequentially to manage API limits and individual processing.  
    - *Configuration:* Default batch size (usually 1).  
    - *Input:* Array of customer/order objects.  
    - *Output:* Individual items passed downstream one at a time.  
    - *Edge Cases:*  
      - Empty input array.  
      - Batch processing delays causing timeouts.  
    - *Sub-workflow:* None

  - **Clean Number**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Cleans phone numbers by removing all non-numeric characters, combines first and last names into a full name, and prepares cleaned customer data.  
    - *Configuration:*  
      - JS code extracts `customer.phone`, filters non-digit characters, concatenates `first_name` and `last_name`.  
    - *Input:* Single customer/order item.  
    - *Output:* Object with cleaned fields: `customer_id`, `name`, `email`, and `phone`.  
    - *Edge Cases:*  
      - Missing or malformed phone numbers.  
      - Customers without first/last names.  
      - Non-string phone fields.  
    - *Sub-workflow:* None

#### 1.4 Batch Processing

- **Overview:**  
  Sequentially processes each cleaned customer item for WhatsApp validation.

- **Nodes Involved:**  
  - Loop Over Items (output connection to Clean Number, described above)  
  - Clean Number (output connection to Check valid whatsapp number Using Rapiwa)

- *Note:* Already covered in previous block with Loop Over Items and Clean Number nodes.

#### 1.5 WhatsApp Number Verification

- **Overview:**  
  Validates if phone numbers are active on WhatsApp using Rapiwa API.

- **Nodes Involved:**  
  - Check valid whatsapp number Using Rapiwa

- **Node Details:**

  - **Check valid whatsapp number Using Rapiwa**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to Rapiwa API to verify if a phone number exists on WhatsApp.  
    - *Configuration:*  
      - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
      - Method: POST  
      - Body parameter: `number` set dynamically to cleaned phone number (`{{$json.phone}}`)  
      - Authentication: HTTP Bearer token (Rapiwa Bearer Auth credentials)  
    - *Input:* Cleaned customer data with phone number.  
    - *Output:* JSON response containing `data.exists` boolean field indicating WhatsApp presence.  
    - *Edge Cases:*  
      - Invalid or expired bearer token.  
      - API rate limiting or downtime.  
      - Invalid phone number format causing API rejection.  
    - *Sub-workflow:* None

#### 1.6 Conditional Messaging & Logging

- **Overview:**  
  Depending on WhatsApp verification, sends message via Rapiwa API or logs as unverified, and logs all results in Google Sheets.

- **Nodes Involved:**  
  - If  
  - Send Message Using Rapiwa  
  - verified append row in sheet  
  - verified append row in sheet1

- **Node Details:**

  - **If**  
    - *Type:* If (Conditional)  
    - *Role:* Routes flow based on WhatsApp existence check.  
    - *Configuration:*  
      - Condition: `{{$json.data.exists}} === "true"` (strict boolean true check)  
    - *Input:* Rapiwa verification response.  
    - *Output:*  
      - True branch: verified WhatsApp numbers.  
      - False branch: unverified numbers.  
    - *Edge Cases:*  
      - Missing or malformed `data.exists` field.  
    - *Sub-workflow:* None

  - **Send Message Using Rapiwa**  
    - *Type:* HTTP Request  
    - *Role:* Sends WhatsApp message with tracking info to verified customers.  
    - *Configuration:*  
      - URL: `https://app.rapiwa.com/api/send-message`  
      - Method: POST  
      - Body parameters:  
        - `number`: Customer phone number  
        - `message_type`: "text"  
        - `message`: Template personalized with customer name, tracking number, and tracking URL from previous nodes  
      - Authentication: HTTP Bearer token (Rapiwa Bearer Auth credentials)  
    - *Input:* Verified phone numbers and customer/order data.  
    - *Output:* Response from message send API.  
    - *Edge Cases:*  
      - Message send failures due to invalid number or API errors.  
      - Rate limiting or network failures.  
    - *Sub-workflow:* None

  - **verified append row in sheet**  
    - *Type:* Google Sheets  
    - *Role:* Logs customers with verified WhatsApp numbers and sent messages to Google Sheets with status `verified`.  
    - *Configuration:*  
      - Operation: Append row  
      - Document & Sheet: Specified Google Sheet (by ID and gid)  
      - Columns mapped with customer_id, name (note trailing space in field), email, phone, tracking info, status = "verified"  
      - Auth: Google Sheets OAuth2 credentials  
    - *Input:* Data after message sent.  
    - *Output:* Confirmation of row append.  
    - *Edge Cases:*  
      - Google Sheets API permission or quota errors.  
      - Field mapping errors due to missing data.  
    - *Sub-workflow:* None

  - **verified append row in sheet1**  
    - *Type:* Google Sheets  
    - *Role:* Logs customers with unverified WhatsApp numbers to Google Sheets with status `unverified`.  
    - *Configuration:*  
      - Same as above but with status field set to "unverified"  
    - *Input:* Data from false branch of If node.  
    - *Output:* Confirmation of row append.  
    - *Edge Cases:*  
      - Same as above.  
    - *Sub-workflow:* None

#### 1.7 Throttling and Looping

- **Overview:**  
  Adds delay between processing batches to prevent API overload or rate limiting.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - *Type:* Wait node  
    - *Role:* Pauses workflow briefly after each append operation before processing next batch/item.  
    - *Configuration:* Default wait time (not specified in JSON, likely standard delay)  
    - *Input:* After Google Sheets append nodes.  
    - *Output:* Connects back to Loop Over Items to continue processing.  
    - *Edge Cases:*  
      - Wait time too short may cause rate limiting.  
      - Excessive wait causes workflow delays.  
    - *Sub-workflow:* None

---

### 3. Summary Table

| Node Name                        | Node Type               | Functional Role                                 | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
|---------------------------------|-------------------------|------------------------------------------------|-------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Shopify Trigger                 | Shopify Trigger          | Triggers on new fulfillment creation in Shopify | (Webhook trigger start)         | Format Webhook Response Data    | ## 1. Node: Shopify Trigger<br>Purpose:<br>This node monitors your Shopify store for the `fulfillments/create` webhook event. It automatically triggers when an order is marked as fulfilled.<br>When it’s Triggered:<br>Whenever an order is fulfilled in Shopify.<br>Webhook Event: fulfillments/create                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Format Webhook Response Data    | Code                    | Extracts key fulfillment and customer data     | Shopify Trigger               | get all customer data           | ## 2. Node: Format Webhook Response Data<br>Purpose:<br>This node takes the raw data from the Shopify webhook and extracts the key information you need.<br>Data Extracted Includes:<br>- Order ID<br>- Tracking number and tracking link<br>- Product title<br>- Customer name and country<br>- Shop domain                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| get all customer data           | HTTP Request            | Retrieves full order and customer data from Shopify API | Format Webhook Response Data  | Loop Over Items                | ## 1. Node: get all customer data<br>Purpose:<br>- Uses the `order_id` to make a Shopify API request and get full customer details for that order.<br><br>## 2. Node: Loop Over Items<br>Purpose:<br>- Loops through each item (e.g., fulfillment) in the workflow — useful if multiple fulfillments come through at once.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Loop Over Items                 | SplitInBatches          | Processes items one at a time                   | get all customer data          | Clean Number                   | ## 1. Node: get all customer data<br>Purpose:<br>- Uses the `order_id` to make a Shopify API request and get full customer details for that order.<br><br>## 2. Node: Loop Over Items<br>Purpose:<br>- Loops through each item (e.g., fulfillment) in the workflow — useful if multiple fulfillments come through at once.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Clean Number                   | Code                    | Cleans and formats customer phone and name data | Loop Over Items               | Check valid whatsapp number Using Rapiwa | ## 1. Node: Check Valid WhatsApp Number Using Rapiwa<br>Purpose:<br>This node uses [Rapiwa’s API](https://rapiwa.com) to verify whether a given phone number is registered and active on WhatsApp.<br><br>## 2. Node: Clean Number<br>Purpose:<br>- Extracts and cleans up the customer’s phone number.<br>- Formats full name from first and last name.<br>- Prepares `customer_id`, email, phone, and name.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Check valid whatsapp number Using Rapiwa | HTTP Request            | Verifies if phone number is WhatsApp active     | Clean Number                  | If                           | ## 1. Node: Check Valid WhatsApp Number Using Rapiwa<br>Purpose:<br>This node uses [Rapiwa’s API](https://rapiwa.com) to verify whether a given phone number is registered and active on WhatsApp.<br><br>## 2. Node: Clean Number<br>Purpose:<br>- Extracts and cleans up the customer’s phone number.<br>- Formats full name from first and last name.<br>- Prepares `customer_id`, email, phone, and name.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| If                            | If                       | Routes based on WhatsApp verification result    | Check valid whatsapp number Using Rapiwa | Send Message Using Rapiwa (true), verified append row in sheet1 (false) | ## 1. Node: If<br>Purpose:<br>Checks the result of the WhatsApp verification:<br>If number exists (i.e., valid WhatsApp number), go one way.<br>If number does not exist, go another way.<br><br>## 2. Node: Send Message Using Rapiwa<br>Purpose:<br>Sends a personalized WhatsApp message to the customer using Rapiwa, with tracking details and a thank-you note.<br><br>## 3. Node: verified append row in sheet<br>Purpose:<br>Logs customer and tracking info to Google Sheets with the status verified.<br><br>## 4. Node: verified append row in sheet1<br>Purpose:<br>Logs the same data to Google Sheets, but with the status unverified.<br><br>## 5. Node: Wait<br>Purpose:<br>Pauses briefly before processing the next item — helps to avoid rate-limiting or API overload. |
| Send Message Using Rapiwa       | HTTP Request            | Sends WhatsApp message with tracking details    | If (true)                    | verified append row in sheet   | ## 1. Node: If<br>Purpose:<br>Checks the result of the WhatsApp verification:<br>If number exists (i.e., valid WhatsApp number), go one way.<br>If number does not exist, go another way.<br><br>## 2. Node: Send Message Using Rapiwa<br>Purpose:<br>Sends a personalized WhatsApp message to the customer using Rapiwa, with tracking details and a thank-you note.<br><br>## 3. Node: verified append row in sheet<br>Purpose:<br>Logs customer and tracking info to Google Sheets with the status verified.<br><br>## 4. Node: verified append row in sheet1<br>Purpose:<br>Logs the same data to Google Sheets, but with the status unverified.<br><br>## 5. Node: Wait<br>Purpose:<br>Pauses briefly before processing the next item — helps to avoid rate-limiting or API overload. |
| verified append row in sheet    | Google Sheets           | Logs verified customers and tracking info       | Send Message Using Rapiwa    | Wait                         | ## 1. Node: If<br>Purpose:<br>Checks the result of the WhatsApp verification:<br>If number exists (i.e., valid WhatsApp number), go one way.<br>If number does not exist, go another way.<br><br>## 2. Node: Send Message Using Rapiwa<br>Purpose:<br>Sends a personalized WhatsApp message to the customer using Rapiwa, with tracking details and a thank-you note.<br><br>## 3. Node: verified append row in sheet<br>Purpose:<br>Logs customer and tracking info to Google Sheets with the status verified.<br><br>## 4. Node: verified append row in sheet1<br>Purpose:<br>Logs the same data to Google Sheets, but with the status unverified.<br><br>## 5. Node: Wait<br>Purpose:<br>Pauses briefly before processing the next item — helps to avoid rate-limiting or API overload. |
| verified append row in sheet1   | Google Sheets           | Logs unverified customers and tracking info     | If (false)                  | Wait                         | ## 1. Node: If<br>Purpose:<br>Checks the result of the WhatsApp verification:<br>If number exists (i.e., valid WhatsApp number), go one way.<br>If number does not exist, go another way.<br><br>## 2. Node: Send Message Using Rapiwa<br>Purpose:<br>Sends a personalized WhatsApp message to the customer using Rapiwa, with tracking details and a thank-you note.<br><br>## 3. Node: verified append row in sheet<br>Purpose:<br>Logs customer and tracking info to Google Sheets with the status verified.<br><br>## 4. Node: verified append row in sheet1<br>Purpose:<br>Logs the same data to Google Sheets, but with the status unverified.<br><br>## 5. Node: Wait<br>Purpose:<br>Pauses briefly before processing the next item — helps to avoid rate-limiting or API overload. |
| Wait                          | Wait                    | Delays to prevent API rate limiting              | verified append row in sheet, verified append row in sheet1 | Loop Over Items               | ## 1. Node: If<br>Purpose:<br>Checks the result of the WhatsApp verification:<br>If number exists (i.e., valid WhatsApp number), go one way.<br>If number does not exist, go another way.<br><br>## 2. Node: Send Message Using Rapiwa<br>Purpose:<br>Sends a personalized WhatsApp message to the customer using Rapiwa, with tracking details and a thank-you note.<br><br>## 3. Node: verified append row in sheet<br>Purpose:<br>Logs customer and tracking info to Google Sheets with the status verified.<br><br>## 4. Node: verified append row in sheet1<br>Purpose:<br>Logs the same data to Google Sheets, but with the status unverified.<br><br>## 5. Node: Wait<br>Purpose:<br>Pauses briefly before processing the next item — helps to avoid rate-limiting or API overload. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger Node**  
   - Type: Shopify Trigger  
   - Event: `fulfillments/create`  
   - Authentication: Configure with Shopify Access Token API credentials  
   - This node will start the workflow on new fulfillment creation.

2. **Add Code Node: Format Webhook Response Data**  
   - Type: Code  
   - JavaScript code to extract essential data fields from the Shopify webhook payload, including order_id, fulfillment_id, tracking info, product title, customer name, country, and shop domain.  
   - Connect Shopify Trigger output to this node.

3. **Add HTTP Request Node: get all customer data**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://your_domain.myshopify.com/admin/api/2025-07/orders/{{ $json.order_id }}.json` (replace `your_domain` and ensure dynamic `order_id` usage)  
   - Headers: Add `X-Shopify-Access-Token` with your Shopify API token  
   - Connect the output of Format Webhook Response Data to this node.  
   - This retrieves full order and customer details.

4. **Add SplitInBatches Node: Loop Over Items**  
   - Type: SplitInBatches  
   - Default batch size (usually 1)  
   - Connect output of get all customer data to this node.  
   - This processes each order/customer sequentially.

5. **Add Code Node: Clean Number**  
   - Type: Code  
   - JavaScript code to:  
     - Extract customer phone number, remove all non-digit characters.  
     - Combine first and last names into full name string.  
     - Prepare JSON with `customer_id`, `name`, `email`, and cleaned `phone`.  
   - Connect Loop Over Items (second output) to this node.

6. **Add HTTP Request Node: Check valid whatsapp number Using Rapiwa**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Body parameter: `number` set to `{{$json.phone}}`  
   - Authentication: HTTP Bearer Auth (set Rapiwa Bearer token credentials)  
   - Connect Clean Number node output to this node.

7. **Add If Node**  
   - Type: If  
   - Condition: Check if `{{$json.data.exists}} === "true"`  
   - Connect output of Check valid whatsapp number Using Rapiwa to this node.

8. **Add HTTP Request Node: Send Message Using Rapiwa** (True branch of If)  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Body parameters:  
     - `number`: customer phone number from data  
     - `message_type`: "text"  
     - `message`: Use template:  
       ```
       Hi {{customer_name}},
       Good news! Your order has just been fulfilled.

       Tracking Number: *{{tracking_number}}*

       Track your package here: *{{tracking_url}}*

       Thank you for shopping with us.
       -Team SpaGreen Creative
       ```  
     - Use expressions to reference previous nodes for dynamic data (customer name, tracking number, URL).  
   - Authentication: HTTP Bearer Auth (Rapiwa token)  
   - Connect If node's true output to this node.

9. **Add Google Sheets Node: verified append row in sheet** (after Send Message)  
   - Type: Google Sheets  
   - Operation: Append row  
   - Document ID and Sheet Name: Point to your Google Sheet with columns: `customer_id`, `name ` (note trailing space), `email`, `number`, `tracking_company`, `tracking_number`, `tracking_url`, `product_title`, `status`  
   - Map data accordingly, set `status` to "verified"  
   - Connect Send Message Using Rapiwa output to this node.

10. **Add Google Sheets Node: verified append row in sheet1** (False branch of If)  
    - Same configuration as above, but set `status` to "unverified"  
    - Connect If node's false output to this node.

11. **Add Wait Node**  
    - Type: Wait  
    - Default pause time (e.g., a few seconds) to avoid API rate limits  
    - Connect outputs of both Google Sheets nodes (verified and unverified) to Wait node.

12. **Loop Back**  
    - Connect Wait node output back to Loop Over Items node to process next batch.

13. **Configure Credentials:**  
    - Shopify Access Token API credentials for Shopify Trigger and HTTP Request nodes.  
    - Rapiwa Bearer Auth (HTTP Bearer Auth type) for Rapiwa API calls.  
    - Google Sheets OAuth2 credentials with write access to the specified spreadsheet.

14. **Validation and Testing:**  
    - Test Shopify webhook by fulfilling an order.  
    - Monitor logs for errors in API calls, formatting, or Google Sheets.  
    - Adjust Wait node timing if rate limits occur.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                                                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses Shopify API version 2025-07, ensure to update if Shopify releases breaking changes in future.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Shopify API documentation: https://shopify.dev/api                                                                                                                                                                                |
| Rapiwa is an unofficial WhatsApp Business API provider; delivery success rates may vary and are not guaranteed. Ensure compliance with WhatsApp terms of service and customer privacy laws.                                                                                                                                                                                                                                                                                                                                                                                                                                          | Rapiwa Official Website: https://rapiwa.com; Documentation: https://docs.rapiwa.com                                                                                                                                                |
| The Google Sheet must include a column named `name ` with a trailing space; do not remove or rename this column as it is referenced directly in the workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sample Google Sheet: https://docs.google.com/spreadsheets/d/1vxocktoY-y-PYBZNxmUDuQv02b5F8QKhbQ0yLHjwjBY/edit?usp=sharing                                                                                                         |
| The Wait node is critical to avoid API rate limits; adjust wait duration based on your volume and API response times.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                                                                                                                    |
| For support or community help, you may use the following: WhatsApp Chat: https://wa.me/8801322827799, Discord SpaGreen Community: https://discord.gg/SsCChWEP, Facebook Group SpaGreen Support: https://www.facebook.com/groups/spagreenbd                                                                                                                |                                                                                                                                                                                                                                    |
| Developer portfolio and additional resources: https://codecanyon.net/user/spagreen/portfolio                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |                                                                                                                                                                                                                                    |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow built with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.