Automated WhatsApp Upsell Messages for Shopify Cancellations with Rapiwa & Google Sheets

https://n8nworkflows.xyz/workflows/automated-whatsapp-upsell-messages-for-shopify-cancellations-with-rapiwa---google-sheets-10790


# Automated WhatsApp Upsell Messages for Shopify Cancellations with Rapiwa & Google Sheets

### 1. Workflow Overview

This workflow automates sending personalized WhatsApp upsell messages to customers who have cancelled their orders on a Shopify store. It targets Shopify merchants aiming to recover cancelled sales by engaging customers with reorder incentives via WhatsApp, verified through the Rapiwa API, and tracks message delivery status in Google Sheets.

The workflow is logically structured into the following blocks:

- **1.1 Scheduled Fetch of Cancelled Orders:** Periodically retrieve all cancelled orders from Shopify.
- **1.2 Customer Order and Profile Data Enrichment:** For each cancelled order, fetch detailed customer orders and profile data, and aggregate product purchase information.
- **1.3 WhatsApp Number Verification via Rapiwa:** Verify if customers’ phone numbers are registered on WhatsApp.
- **1.4 Personalized Upsell Message Dispatch:** Send customized WhatsApp messages with reorder links and discount coupons to verified numbers.
- **1.5 State Recording in Google Sheets:** Log the status of message attempts (sent/verified or unverified/not sent) in a Google Sheet for tracking and analytics.
- **1.6 Loop and Wait Control:** Manage iteration over multiple customers and pacing of message sends.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Fetch of Cancelled Orders

- **Overview:** Automatically triggers on a schedule to fetch all Shopify orders with a cancelled status.
- **Nodes Involved:** `Schedule Trigger`, `Get All Cancelled Order`, `Split Out`, `Loop Over Items`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Initiates the workflow periodically (default interval: every minute)
    - Inputs: None
    - Outputs: Triggers `Get All Cancelled Order`
    - Edge Cases: Misconfigured schedule could cause missed or too frequent runs.

  - **Get All Cancelled Order**
    - Type: HTTP Request
    - Role: Calls Shopify API to fetch all cancelled orders (`status=cancelled`)
    - Configuration: Uses Shopify Admin API endpoint with access token header; URL: `https://your_shopify_domain/admin/api/2025-07/orders.json?customer_id=&status=cancelled`
    - Inputs: Triggered by Schedule Trigger
    - Outputs: JSON containing cancelled orders list to `Split Out`
    - Failures: Network errors, API rate limits, invalid access token.

  - **Split Out**
    - Type: Split Out
    - Role: Extracts the `orders` array from API response to separate items for batch processing
    - Configuration: Splits by the field `orders`
    - Inputs: JSON object with `orders` array from `Get All Cancelled Order`
    - Outputs: Individual order items to `Loop Over Items`
    - Edge Cases: If `orders` field missing or empty, no items processed.

  - **Loop Over Items**
    - Type: Split In Batches
    - Role: Controls batch processing of each cancelled order item
    - Configuration: Default batching with no specific batch size configured
    - Inputs: Individual orders from `Split Out`
    - Outputs: Processes each order item onward to customer-specific data fetching
    - Edge Cases: Large data sets may cause long processing times.

---

#### 2.2 Customer Order and Profile Data Enrichment

- **Overview:** For each cancelled order’s customer, fetch all their orders, extract product and spending info, then retrieve detailed customer profile data.
- **Nodes Involved:** `Get Specific Customer Data`, `Get Product name with re-order llink`, `Get customer info`, `Get Customer Total Spent`
- **Node Details:**

  - **Get Specific Customer Data**
    - Type: HTTP Request
    - Role: Fetches all orders for a specific customer, regardless of status
    - Configuration: Dynamic URL with customer ID from previous node; Shopify API endpoint: `/orders.json?customer_id={{ $json.customer.id }}&status=any`
    - Inputs: Customer ID from `Loop Over Items`
    - Outputs: Orders for the customer to `Get Product name with re-order llink`
    - Failures: Missing/invalid customer ID, API errors.

  - **Get Product name with re-order llink**
    - Type: Code (JavaScript)
    - Role: Processes the customer orders list to aggregate:
      - Customer basic info (name, phone, email)
      - Total spent
      - Products purchased with quantities
      - Last order status URL (used as reorder link)
    - Configuration: Custom JS code that validates array, builds a map keyed by customer ID, then outputs structured data per customer
    - Inputs: Orders array from `Get Specific Customer Data`
    - Outputs: Structured customer-product data to `Get customer info`
    - Edge Cases: Missing orders array, malformed data, no line items.

  - **Get customer info**
    - Type: HTTP Request
    - Role: Retrieves detailed customer profile data from Shopify using customer ID
    - Configuration: Dynamic URL: `/customers/{{ $json.customerId }}.json`
    - Inputs: Customer ID from `Get Product name with re-order llink`
    - Outputs: Customer JSON data to `Get Customer Total Spent`
    - Failures: Invalid customer IDs, API errors.

  - **Get Customer Total Spent**
    - Type: Code (JavaScript)
    - Role: Filters customers who have spent more than zero; extracts and formats customer details such as phone, email, address, and timestamps
    - Inputs: Customer data JSON from `Get customer info`
    - Outputs: Filtered and cleaned customer data to `Rapiwa (verify number)`
    - Edge Cases: Missing customer data, zero spending customers are filtered out.

---

#### 2.3 WhatsApp Number Verification via Rapiwa

- **Overview:** Verifies whether the customer's phone number is registered on WhatsApp using the Rapiwa API.
- **Nodes Involved:** `Rapiwa (verify number)`, `If`
- **Node Details:**

  - **Rapiwa (verify number)**
    - Type: Rapiwa API Node
    - Role: Checks if phone number is WhatsApp-registered
    - Configuration: Parameter `number` is dynamically set from customer phone field
    - Credentials: Requires Rapiwa API key (configured)
    - Inputs: Customer phone data from `Get Customer Total Spent`
    - Outputs: Verification result to `If` node
    - Failures: API key invalid, rate limit, network errors.

  - **If**
    - Type: Conditional (If)
    - Role: Branches workflow based on whether the phone number is verified as WhatsApp-enabled
    - Configuration: Checks boolean `exists` field from Rapiwa response equals `true`
    - Inputs: Verification response from `Rapiwa (verify number)`
    - Outputs:
      - True branch (verified): to `Rapiwa (sent message)`
      - False branch (unverified): to `Store State of Rows in Unverified & Not Sent`
    - Edge Cases: Missing `exists` field, unexpected response format.

---

#### 2.4 Personalized Upsell Message Dispatch

- **Overview:** Sends a customized WhatsApp message offering a reorder discount link to verified WhatsApp numbers.
- **Nodes Involved:** `Rapiwa (sent message)`
- **Node Details:**

  - **Rapiwa (sent message)**
    - Type: Rapiwa API Node
    - Role: Sends a WhatsApp message to the verified number
    - Configuration:
      - `number`: Uses the `jid` returned from verification node (WhatsApp ID)
      - `message`: Template message includes customer name, reorder link, and discount coupon `REORDER5`
    - Credentials: Uses the configured Rapiwa API key
    - Inputs: Customer WhatsApp ID and details from prior nodes
    - Outputs: Confirmation to `Store State of Rows in Verified & Sent`
    - Failures: Message sending failures, invalid JID, API limits.

---

#### 2.5 State Recording in Google Sheets

- **Overview:** Records the status of each message attempt (sent/verified or unverified/not sent) in Google Sheets for tracking.
- **Nodes Involved:** `Store State of Rows in Verified & Sent`, `Store State of Rows in Unverified & Not Sent`, `Wait`
- **Node Details:**

  - **Store State of Rows in Verified & Sent**
    - Type: Google Sheets
    - Role: Appends a new row to a Google Sheet to track successfully sent messages with validity `verified` and status `sent`
    - Configuration: Defines columns such as name, phone number, coupon, item link, item name, status, and validity
    - Credentials: Google Sheets OAuth2 credential required
    - Inputs: Message details from `Rapiwa (sent message)`
    - Outputs: To `Wait` node
    - Edge Cases: Sheet access permission, quota limits.

  - **Store State of Rows in Unverified & Not Sent**
    - Type: Google Sheets
    - Role: Appends a row for numbers that failed verification (validity `unverified`) and status `sent` (message not sent)
    - Configuration: Same columns as above
    - Credentials: Google Sheets OAuth2 credential required
    - Inputs: From `If` node false branch
    - Outputs: To `Wait` node
    - Edge Cases: Same as above.

  - **Wait**
    - Type: Wait
    - Role: Introduces delay or pacing between processing batches to avoid API rate limits or flooding
    - Configuration: Default wait configuration (no duration specified, but webhook ID present)
    - Inputs: From both Google Sheets nodes
    - Outputs: Loops back to `Loop Over Items` to process next batch
    - Edge Cases: Misconfigured wait timing could cause delays or rapid firing.

---

### 3. Summary Table

| Node Name                              | Node Type               | Functional Role                                  | Input Node(s)                    | Output Node(s)                        | Sticky Note                                                                                                   |
|--------------------------------------|-------------------------|-------------------------------------------------|---------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                     | Schedule Trigger        | Initiates workflow periodically                  | None                            | Get All Cancelled Order              |                                                                                                              |
| Get All Cancelled Order              | HTTP Request            | Fetches all cancelled Shopify orders             | Schedule Trigger                | Split Out                           | "Get All Cancelled Order" - Purpose: Fetches all cancelled orders from Shopify                               |
| Split Out                          | Split Out               | Extracts orders array for processing              | Get All Cancelled Order         | Loop Over Items                    |                                                                                                              |
| Loop Over Items                    | Split In Batches        | Iterates over each cancelled order item           | Split Out                      | Get Specific Customer Data          |                                                                                                              |
| Get Specific Customer Data         | HTTP Request            | Retrieves all orders for a specific customer      | Loop Over Items                | Get Product name with re-order llink | "Get Specific Customer Data" - Purpose: fetches all orders for specific customer                             |
| Get Product name with re-order llink | Code (JavaScript)       | Aggregates customer and product purchase info     | Get Specific Customer Data      | Get customer info                  | "Get Product Name with Re-order Link" - Purpose: classify orders and extract useful details                  |
| Get customer info                  | HTTP Request            | Retrieves detailed customer profile data          | Get Product name with re-order llink | Get Customer Total Spent          |                                                                                                              |
| Get Customer Total Spent           | Code (JavaScript)       | Filters customers with >0 spend, formats data     | Get customer info              | Rapiwa (verify number)             | "Get Customer Total Spent" - Purpose: filters customers who have spent 0 out                                |
| Rapiwa (verify number)             | Rapiwa API Node         | Verifies if phone number is WhatsApp registered   | Get Customer Total Spent       | If                                | "Check valid WhatsApp number Using Rapiwa" - Purpose: verify WhatsApp registration                            |
| If                                | Conditional (If)        | Branches on WhatsApp verification result          | Rapiwa (verify number)         | Rapiwa (sent message), Store State of Rows in Unverified & Not Sent |                                                                                                              |
| Rapiwa (sent message)              | Rapiwa API Node         | Sends personalized WhatsApp upsell message        | If (true branch)               | Store State of Rows in Verified & Sent | "Rapiwa Sender" - Purpose: send personalized WhatsApp messages for verified numbers                         |
| Store State of Rows in Verified & Sent | Google Sheets           | Logs successfully sent messages with verified status | Rapiwa (sent message)          | Wait                              | "Store State of Rows in Verified & Sent" - references a Google spreadsheet                                  |
| Store State of Rows in Unverified & Not Sent | Google Sheets           | Logs unverified numbers and unsent messages       | If (false branch)              | Wait                              | "Store State of Rows in Unverified & Not Sent" - writes rows for numbers not registered on WhatsApp         |
| Wait                              | Wait                    | Controls pacing and loops workflow                 | Store State of Rows in Verified & Sent, Store State of Rows in Unverified & Not Sent | Loop Over Items                    |                                                                                                              |
| Sticky Note (6)                   | Sticky Note             | Documentation block covering Rapiwa verification and sending, Google Sheets tracking | None                            | None                              | Explains Rapiwa WhatsApp verification, message sending, and Google Sheets tracking blocks                   |
| Sticky Note (1)                   | Sticky Note             | Documentation for "Get All Cancelled Order" node | None                            | None                              | Summarizes purpose of fetching cancelled orders from Shopify                                               |
| Sticky Note (2)                   | Sticky Note             | Documentation for "Get Customer Total Spent" node | None                            | None                              | Explains filtering customers who spent zero                                                            |
| Sticky Note (3)                   | Sticky Note             | Documentation for "Get Specific Customer Data" and "Get Product name with re-order llink" nodes | None                            | None                              | Explains purpose of fetching specific customer orders and product extraction                               |
| Sticky Note (Main Summary)        | Sticky Note             | High-level overview, workflow steps, useful links | None                            | None                              | Detailed workflow overview, Shopify API docs, Rapiwa setup, and Google Sheet sample link                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Configuration: Default interval (e.g., every 1 minute)
   - Purpose: Start workflow regularly to fetch cancelled orders

2. **Add HTTP Request Node: Get All Cancelled Order**
   - Connect from Schedule Trigger
   - Method: GET
   - URL: `https://your_shopify_domain/admin/api/2025-07/orders.json?customer_id=&status=cancelled`
   - Headers: Add header `X-Shopify-Access-Token` with your Shopify access token
   - Purpose: Fetch all cancelled orders

3. **Add Split Out Node**
   - Connect from Get All Cancelled Order
   - Field to split out: `orders`
   - Purpose: Break the response into individual order items

4. **Add Split In Batches Node: Loop Over Items**
   - Connect from Split Out
   - Default batch size (or specify as needed)
   - Purpose: Process each cancelled order individually

5. **Add HTTP Request Node: Get Specific Customer Data**
   - Connect from Loop Over Items (second output)
   - Method: GET
   - URL Template: `https://your_shopify_domain/admin/api/2025-07/orders.json?customer_id={{ $json.customer.id }}&status=any`
   - Headers: `X-Shopify-Access-Token` with your token
   - Purpose: Get all orders for the customer to analyze purchase history

6. **Add Code Node: Get Product name with re-order llink**
   - Connect from Get Specific Customer Data
   - Paste provided JavaScript code that:
     - Validates orders array
     - Builds customer info with total spent, products, reorder link
   - Purpose: Aggregate customer order data

7. **Add HTTP Request Node: Get customer info**
   - Connect from Code Node
   - Method: GET
   - URL Template: `https://your_shopify_domain/admin/api/2025-07/customers/{{ $json.customerId }}.json`
   - Headers: `X-Shopify-Access-Token`
   - Purpose: Fetch detailed customer profile info

8. **Add Code Node: Get Customer Total Spent**
   - Connect from Get customer info
   - Paste provided JavaScript code that:
     - Filters customers with total spent > 0
     - Extracts and formats customer details
   - Purpose: Prepare customer data for WhatsApp verification

9. **Add Rapiwa Node: Rapiwa (verify number)**
   - Connect from Get Customer Total Spent
   - Operation: `verifyWhatsAppNumber`
   - Parameter: `number` from customer phone field (`={{ $json.phone }}`)
   - Credentials: Set Rapiwa API key
   - Purpose: Verify WhatsApp registration of phone

10. **Add Conditional Node: If**
    - Connect from Rapiwa (verify number)
    - Condition: `{{ $json.exists }} == true`
    - Purpose: Branch workflow by WhatsApp verification result

11. **Add Rapiwa Node: Rapiwa (sent message)**
    - Connect from If node true branch
    - Operation: Send WhatsApp message
    - Parameters:
      - `number`: Use `={{ $json.jid }}`
      - `message`: Template including customer name, reorder link, coupon code `REORDER5`
    - Credentials: Same Rapiwa API key
    - Purpose: Send upsell WhatsApp message

12. **Add Google Sheets Node: Store State of Rows in Verified & Sent**
    - Connect from Rapiwa (sent message)
    - Operation: Append row
    - Document ID: Your Google Sheet ID
    - Sheet Name: `Sheet1` or appropriate gid
    - Columns: Map fields such as name, number, coupon, status `sent`, validity `verified`, item link, item name
    - Credentials: Google Sheets OAuth2 credentials
    - Purpose: Log sent and verified messages

13. **Add Google Sheets Node: Store State of Rows in Unverified & Not Sent**
    - Connect from If node false branch
    - Operation: Append row
    - Same Google Sheet and columns as above, but status `sent` and validity `unverified`
    - Credentials: Google Sheets OAuth2 credentials
    - Purpose: Log unverified (WhatsApp number not found) attempts

14. **Add Wait Node**
    - Connect from both Google Sheets nodes
    - Purpose: Introduce delay before next batch processing
    - Configuration: Default or custom wait time as desired

15. **Connect Wait Node back to Loop Over Items**
    - Purpose: Loop to process next customer/order

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow uses the Rapiwa API to verify WhatsApp numbers and send messages.                                         | Rapiwa API Docs: https://docs.rapiwa.com                                                             |
| Google Sheets is used to track the status of message attempts for analytics and auditing.                               | Sample Sheet: https://docs.google.com/spreadsheets/d/1nvrxINR5Cch5SChGydf62W6PcDdTmdDnlqBif6iTgLU/edit |
| Shopify Admin API version 2025-07 is used; ensure API keys and permissions are valid and scopes include orders/customers. | Shopify API Docs: https://shopify.dev/docs/admin-api                                                 |
| Rapiwa Node installation link: [How to install Rapiwa](https://www.npmjs.com/package/n8n-nodes-rapiwa)                  |                                                                                                     |
| Rapiwa Dashboard for account management: https://app.rapiwa.com/login                                                  |                                                                                                     |
| The workflow includes thorough error handling for missing data and type validation in code nodes.                      |                                                                                                     |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.