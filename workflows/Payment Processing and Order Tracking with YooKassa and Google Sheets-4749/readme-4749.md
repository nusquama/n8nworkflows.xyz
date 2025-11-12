Payment Processing and Order Tracking with YooKassa and Google Sheets

https://n8nworkflows.xyz/workflows/payment-processing-and-order-tracking-with-yookassa-and-google-sheets-4749


# Payment Processing and Order Tracking with YooKassa and Google Sheets

### 1. Workflow Overview

This workflow enables online payment processing and order tracking by integrating YooKassa payment API with Google Sheets for product listing, order logging, and transaction management. It supports multiple HTTP endpoints: listing products, initiating payments, handling YooKassa webhooks for payment/refund events, and querying payment status. The workflow is designed for simple e-commerce or digital product sales scenarios without requiring backend code.

The workflow is logically divided into the following blocks:

- **1.1 Product Listing API**: Handles `GET /products` requests, fetching product data from Google Sheets, sorting by price, and returning the list.

- **1.2 Payment Initiation API**: Handles `POST /payment` requests, validating input, fetching product details, generating an idempotence key, creating a YooKassa payment, saving the order in Google Sheets, and responding with a confirmation URL.

- **1.3 YooKassa Webhook Processing**: Handles payment and refund events posted by YooKassa to update transactions in Google Sheets accordingly.

- **1.4 Payment Status API**: Handles `GET /status/:id` requests to query YooKassa for current payment status and return it.

- **1.5 Error Handling**: Dedicated nodes respond with appropriate HTTP codes and messages upon validation failures or external API errors.

The workflow also includes multiple sticky notes documenting sections and summarizing the workflow purpose and usage.

---

### 2. Block-by-Block Analysis

#### 1.1 Product Listing API

- **Overview:** Provides a sorted list of products from a Google Sheets document via a webhook endpoint `/products`.

- **Nodes Involved:**
  - Get products (Webhook)
  - Get products_1 (Google Sheets)
  - Sorting by price (Sort)
  - Respond Products (Respond to Webhook)
  - Handle Error (get product) (Respond to Webhook)
  - Sticky Note5 (Documentation)

- **Node Details:**

  - **Get products**
    - Type: Webhook (HTTP GET)
    - Configured path: `/products`
    - Response mode: Uses response node to return data
    - Input: HTTP request
    - Output: Passes to Google Sheets node to fetch products
    - Edge cases: Missing sheet data or Google Sheets API errors handled downstream

  - **Get products_1**
    - Type: Google Sheets (Read)
    - Reads all rows from the `products` sheet (sheetId = 0) in a specified Google Sheets document
    - Credentials: Google Sheets OAuth2
    - Retries twice on failure
    - Input: From webhook node
    - Output: Product data list to be sorted

  - **Sorting by price**
    - Type: Sort
    - Sorts the product list by the field `price` ascending
    - Input: Product data from Google Sheets
    - Output: Sorted list forwarded to response

  - **Respond Products**
    - Type: Respond to Webhook
    - Returns all incoming items (the sorted product list)
    - Input: Sorted data
    - Output: HTTP response to the client

  - **Handle Error (get product)**
    - Type: Respond to Webhook
    - Returns HTTP 400 with no data if an error occurs in fetching products
    - Connected as error fallback for Google Sheets node

  - **Sticky Note5**
    - Emphasizes the "Get Products" section for clarity

- **Edge Cases:**
  - Google Sheets API failures or empty product list
  - Invalid webhook requests (unlikely for GET)
  - Network timeouts

---

#### 1.2 Payment Initiation API

- **Overview:** Accepts payment requests via `/payment` POST, validates inputs, fetches product details, generates an idempotence key, calls YooKassa API to create payment, logs order in Google Sheets, and returns payment confirmation URL.

- **Nodes Involved:**
  - Payment (Webhook)
  - Check request values (If)
  - Email validation (If)
  - Get product_id (Set)
  - Get Product (Google Sheets)
  - Check product_id (If)
  - Idempotence Key Generation (Code)
  - YooKassa Request (HTTP Request)
  - Save Order (Google Sheets)
  - Respond Payment (Respond to Webhook)
  - Handle Error, Handle Error (product_id), Handle Error (yookassa), Handle Error (save order) (Respond to Webhook)
  - Sticky Note (Documentation)

- **Node Details:**

  - **Payment**
    - Type: Webhook (HTTP POST)
    - Path: `/payment`
    - Raw body enabled to access JSON input
    - Output: Passes request body to validation node

  - **Check request values**
    - Type: If
    - Validates existence of required fields in request body: `product_id`, `email`, `return_url`
    - Only passes forward if all exist
    - Output: Valid or invalid paths

  - **Email validation**
    - Type: If
    - Validates email format using regex on the `email` field from request body
    - Output: Valid or invalid paths

  - **Get product_id**
    - Type: Set
    - Extracts `product_id` from request body into a variable for next steps

  - **Get Product**
    - Type: Google Sheets (Read)
    - Filters rows in `products` sheet by `product_id`
    - Credentials: Google Sheets OAuth2
    - Max 2 retries on failure
    - Output: Product data if found

  - **Check product_id**
    - Type: If
    - Checks if product data exists from previous node output
    - If product not found, triggers error response

  - **Idempotence Key Generation**
    - Type: Code
    - Generates a random UUID v4 string for idempotence in API request
    - Output: UUID string in `data` field

  - **YooKassa Request**
    - Type: HTTP Request
    - Calls YooKassa API `POST https://api.yookassa.ru/v3/payments` to create payment
    - Authentication: HTTP Basic Auth with YooKassa credentials (shopId/secretKey)
    - Uses idempotence key in header for safe retries
    - Constructs payment JSON body with product price, email, return URL, and description
    - Output: YooKassa payment response with confirmation URL

  - **Save Order**
    - Type: Google Sheets (Append)
    - Logs order details (`id`, `email`, `product_id`, `order_datetime`) in `orders` sheet
    - Credentials: Google Sheets OAuth2
    - Retries enabled
    - Output: Success or failure

  - **Respond Payment**
    - Type: Respond to Webhook
    - Returns HTTP 200 with JSON containing YooKassa payment confirmation URL

  - **Handle Error nodes**
    - Various Respond to Webhook nodes returning HTTP 400 or 404 with error messages for validation failures, missing product, YooKassa request failure, or order saving failure

  - **Sticky Note**
    - Labels the "Payment" block for clarity

- **Edge Cases:**
  - Missing or malformed request fields
  - Email regex validation failure
  - Product not found in Google Sheets
  - YooKassa API authentication or network errors
  - Duplicate requests handled by idempotence key
  - Google Sheets write failures when saving orders

---

#### 1.3 YooKassa Webhook Processing

- **Overview:** Handles POST requests to `/yoomoney` webhook for YooKassa events - primarily `payment.succeeded` and `refund.succeeded`. Updates transaction logs in Google Sheets accordingly.

- **Nodes Involved:**
  - Handle YooKassa webhook (Webhook)
  - Handle Events (Switch)
  - Validation metadata (If)
  - Get Order (Google Sheets)
  - Check Order (If)
  - Save Transaction (Google Sheets)
  - Validation metadata_ (If)
  - Get transaction (Google Sheets)
  - Check if transaction exists (If)
  - Update refund status (Google Sheets)
  - Respond payment event (Respond to Webhook)
  - Respond refund event (Respond to Webhook)
  - Handle Error webhook (Respond to Webhook)
  - Sticky Note1 (Documentation)

- **Node Details:**

  - **Handle YooKassa webhook**
    - Type: Webhook (HTTP POST)
    - Path: `/yoomoney`
    - Raw body enabled to parse JSON event from YooKassa

  - **Handle Events**
    - Type: Switch
    - Routes events based on `$json.body.event` field
    - Handles `payment.succeeded` and `refund.succeeded` events

  - **Validation metadata** (for `payment.succeeded`)
    - Type: If
    - Checks existence of payment `id` and metadata email in event object

  - **Get Order**
    - Type: Google Sheets (Read)
    - Looks up order by `id` from event object in `orders` sheet

  - **Check Order**
    - Type: If
    - Verifies the order was found

  - **Save Transaction**
    - Type: Google Sheets (Append)
    - Logs transaction with payment ID, email, product_id, status "succeeded", and captured timestamp in `transactions` sheet

  - **Validation metadata_** (for `refund.succeeded`)
    - Type: If
    - Checks existence of payment `id` in event object

  - **Get transaction**
    - Type: Google Sheets (Read)
    - Looks up transaction by payment ID in `transactions` sheet

  - **Check if transaction exists**
    - Type: If
    - Verifies the transaction exists

  - **Update refund status**
    - Type: Google Sheets (Update)
    - Updates transaction status to "refund" for the matching payment ID

  - **Respond payment event**
    - Type: Respond to Webhook
    - Returns HTTP 200 with no content on successful payment event processing

  - **Respond refund event**
    - Type: Respond to Webhook
    - Returns HTTP 200 with no content on successful refund event processing

  - **Handle Error webhook**
    - Type: Respond to Webhook
    - Returns HTTP 200 with no content on error to silently acknowledge webhook

  - **Sticky Note1**
    - Labels the "Webhook" section

- **Edge Cases:**
  - Missing or malformed webhook event data
  - Order or transaction not found in Google Sheets
  - Google Sheets API errors or timeouts during reads/updates
  - Idempotency: No retries needed on webhook, but ensures no duplicate logs
  - Other YooKassa event types ignored

---

#### 1.4 Payment Status API

- **Overview:** Provides a GET endpoint `/status/:id` to fetch and return payment status from YooKassa API based on payment ID.

- **Nodes Involved:**
  - Status (Webhook)
  - Check request values_1 (If)
  - YooKassa Request Status (HTTP Request)
  - Respond status (Respond to Webhook)
  - Handle Error status (Respond to Webhook)
  - Handle Error (check request) (Respond to Webhook)
  - Sticky Note3 (Documentation)

- **Node Details:**

  - **Status**
    - Type: Webhook (HTTP GET)
    - Path: `/status/:id` with URL parameter `id`

  - **Check request values_1**
    - Type: If
    - Validates presence of `id` URL parameter

  - **YooKassa Request Status**
    - Type: HTTP Request
    - Calls YooKassa API `GET https://api.yookassa.ru/v3/payments/{id}`
    - Authentication: HTTP Basic Auth with YooKassa credentials

  - **Respond status**
    - Type: Respond to Webhook
    - Returns JSON with payment `id`, `status`, and `description` fields

  - **Handle Error status**
    - Type: Respond to Webhook
    - Returns error status code and message if YooKassa API call fails

  - **Handle Error (check request)**
    - Type: Respond to Webhook
    - Returns HTTP 400 if `id` is missing in request

  - **Sticky Note3**
    - Labels the "Status" section

- **Edge Cases:**
  - Missing `id` parameter
  - YooKassa API errors or invalid payment ID
  - Network or authentication failures

---

#### 1.5 General Error Handling

- **Overview:** Throughout the workflow, dedicated respond-to-webhook nodes return appropriate HTTP status codes (400, 404) and messages for various validation or external API errors.

- **Nodes Involved:**
  - Handle Error (get product), Handle Error (get product)_1
  - Handle Error (product_id)
  - Handle Error (yookassa)
  - Handle Error (save order)
  - Handle Error (check request)
  - Handle Error (status)
  - Handle Error webhook
  - Handle Error

- **Node Details:**
  - These nodes are connected as fallback/error handlers for their respective blocks.
  - They provide clear feedback for debugging and client integration.
  - Returning no data or text messages as appropriate.

- **Edge Cases:**
  - Ensures client receives meaningful HTTP responses on failures.
  - Prevents workflow halting on errors by using continueErrorOutput or continueRegularOutput.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                           |
|---------------------------|-------------------------|----------------------------------------|-------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Get products              | Webhook                 | Entry point for `/products` GET API    | -                             | Get products_1                 | # Get Products                                                                                                        |
| Get products_1            | Google Sheets           | Reads products sheet                    | Get products                  | Sorting by price               | # Get Products                                                                                                        |
| Sorting by price          | Sort                    | Sorts products by price                 | Get products_1                | Respond Products               | # Get Products                                                                                                        |
| Respond Products          | Respond to Webhook      | Returns sorted product list             | Sorting by price              | -                              | # Get Products                                                                                                        |
| Handle Error (get product)| Respond to Webhook      | Error handler for getting products     | Get products_1                | -                              | # Get Products                                                                                                        |
| Payment                  | Webhook                 | Entry point for `/payment` POST API    | -                             | Check request values           | # Payment                                                                                                            |
| Check request values      | If                      | Validates required payment request data| Payment                      | Email validation / Handle Error| # Payment                                                                                                            |
| Email validation          | If                      | Validates email format                  | Check request values          | Get product_id / Handle Error  | # Payment                                                                                                            |
| Get product_id            | Set                     | Extracts product_id from request       | Email validation              | Get Product                   | # Payment                                                                                                            |
| Get Product               | Google Sheets           | Retrieves product details by product_id| Get product_id               | Check product_id               | # Payment                                                                                                            |
| Check product_id          | If                      | Checks if product is found              | Get Product                  | Idempotence Key Generation / Handle Error (product_id)| # Payment                                                                                                            |
| Idempotence Key Generation| Code                    | Generates UUID for idempotence          | Check product_id             | YooKassa Request              | # Payment                                                                                                            |
| YooKassa Request          | HTTP Request            | Creates payment in YooKassa             | Idempotence Key Generation   | Save Order / Handle Error (yookassa)| # Payment                                                                                                            |
| Save Order                | Google Sheets           | Appends order data to `orders` sheet   | YooKassa Request             | Respond Payment / Handle Error (save order)| # Payment                                                                                                            |
| Respond Payment           | Respond to Webhook      | Returns YooKassa confirmation URL      | Save Order                   | -                              | # Payment                                                                                                            |
| Handle Error              | Respond to Webhook      | Generic bad request response            | Email validation / Check request values| -                     | # Payment                                                                                                            |
| Handle Error (product_id) | Respond to Webhook      | Product not found response              | Check product_id             | -                              | # Payment                                                                                                            |
| Handle Error (yookassa)   | Respond to Webhook      | YooKassa API error response             | YooKassa Request             | -                              | # Payment                                                                                                            |
| Handle Error (save order) | Respond to Webhook      | Order saving error response             | Save Order                   | -                              | # Payment                                                                                                            |
| Handle YooKassa webhook   | Webhook                 | Entry point for YooKassa webhook events| -                             | Handle Events                 | # Webhook                                                                                                            |
| Handle Events             | Switch                  | Routes YooKassa events                   | Handle YooKassa webhook      | Validation metadata / Validation metadata_| # Webhook                                                                                                            |
| Validation metadata       | If                      | Validates payment event metadata        | Handle Events (payment.succeeded)| Get Order / Handle Error webhook| # Webhook                                                                                                            |
| Get Order                 | Google Sheets           | Retrieves order by payment ID           | Validation metadata          | Check Order                   | # Webhook                                                                                                            |
| Check Order               | If                      | Checks if order found                    | Get Order                   | Save Transaction / Handle Error webhook| # Webhook                                                                                                            |
| Save Transaction          | Google Sheets           | Logs successful payment transaction     | Check Order                 | Respond payment event          | # Webhook                                                                                                            |
| Validation metadata_      | If                      | Validates refund event metadata          | Handle Events (refund.succeeded)| Get transaction / Handle Error webhook| # Webhook                                                                                                            |
| Get transaction           | Google Sheets           | Retrieves transaction by payment ID     | Validation metadata_         | Check if transaction exists    | # Webhook                                                                                                            |
| Check if transaction exists| If                     | Checks if transaction exists             | Get transaction             | Update refund status / Handle Error webhook| # Webhook                                                                                                            |
| Update refund status      | Google Sheets           | Updates transaction status to refund    | Check if transaction exists | Respond refund event           | # Webhook                                                                                                            |
| Respond payment event     | Respond to Webhook      | Acknowledges payment webhook event      | Save Transaction            | -                              | # Webhook                                                                                                            |
| Respond refund event      | Respond to Webhook      | Acknowledges refund webhook event       | Update refund status        | -                              | # Webhook                                                                                                            |
| Handle Error webhook      | Respond to Webhook      | Catches errors in webhook processing     | Get Order / Check Order / Validation metadata / Get transaction / Check if transaction exists | - | # Webhook                                                                                                            |
| Status                   | Webhook                 | Entry point for `/status/:id` GET API   | -                             | Check request values_1        | # Status                                                                                                             |
| Check request values_1    | If                      | Validates presence of payment ID param  | Status                      | YooKassa Request Status / Handle Error (check request)| # Status                                                                                                             |
| YooKassa Request Status   | HTTP Request            | Fetches payment status from YooKassa    | Check request values_1       | Respond status / Handle Error status| # Status                                                                                                             |
| Respond status            | Respond to Webhook      | Returns payment status JSON              | YooKassa Request Status      | -                              | # Status                                                                                                             |
| Handle Error status       | Respond to Webhook      | Handles YooKassa status request errors  | YooKassa Request Status      | -                              | # Status                                                                                                             |
| Handle Error (check request)| Respond to Webhook    | Handles missing payment ID errors        | Check request values_1       | -                              | # Status                                                                                                             |
| Sticky Note               | Sticky Note             | Labels "Payment" section                  | -                             | -                              | # Payment                                                                                                            |
| Sticky Note1              | Sticky Note             | Labels "Webhook" section                  | -                             | -                              | # Webhook                                                                                                            |
| Sticky Note3              | Sticky Note             | Labels "Status" section                   | -                             | -                              | # Status                                                                                                             |
| Sticky Note5              | Sticky Note             | Labels "Get Products" section             | -                             | -                              | # Get Products                                                                                                        |
| Sticky Note2              | Sticky Note             | Overview and documentation of entire workflow| -                        | -                              | # Accept YooKassa payments and log transactions in Google Sheets... (full documentation included)                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Get products"**
   - HTTP Method: GET
   - Path: `/products`
   - Response Mode: Response Node

2. **Create Google Sheets Node "Get products_1"**
   - Operation: Read Rows
   - Document ID: Your Google Sheets document with products
   - Sheet Name: `products` (sheetId=0)
   - Credentials: Google Sheets OAuth2
   - Retry: 2 times on failure
   - Connect input from "Get products" node

3. **Create Sort Node "Sorting by price"**
   - Sort Field: `price` ascending
   - Connect input from "Get products_1"

4. **Create Respond to Webhook Node "Respond Products"**
   - Respond With: All Incoming Items
   - Connect input from "Sorting by price"

5. **Create Respond to Webhook Node "Handle Error (get product)"**
   - Response Code: 400
   - Respond With: No Data
   - Connect as error fallback from "Get products_1"

6. **Create Webhook Node "Payment"**
   - HTTP Method: POST
   - Path: `/payment`
   - Options: Enable Raw Body
   - Response Mode: Response Node

7. **Create If Node "Check request values"**
   - Condition: `product_id`, `email`, and `return_url` exist in `$json.body`
   - Connect input from "Payment"

8. **Create If Node "Email validation"**
   - Condition: Validate `$json.body.email` with regex `^[^\s@]+@[^\s@]+\.[^\s@]+$`
   - Connect True output from "Check request values"

9. **Create Set Node "Get product_id"**
   - Set field `product_id` to `{{$json.body.product_id}}`
   - Connect True output from "Email validation"

10. **Create Google Sheets Node "Get Product"**
    - Operation: Read Rows with Filter
    - Filter: `product_id` equals value from `Get product_id`
    - Sheet: `products`
    - Document and credentials same as previous
    - Retry: 2 times
    - Connect input from "Get product_id"

11. **Create If Node "Check product_id"**
    - Condition: Check if Google Sheets output item exists
    - Connect True output from "Get Product"

12. **Create Code Node "Idempotence Key Generation"**
    - JavaScript code generates UUID v4 string for idempotence key
    - Connect True output from "Check product_id"

13. **Create HTTP Request Node "YooKassa Request"**
    - Method: POST
    - URL: `https://api.yookassa.ru/v3/payments`
    - Authentication: HTTP Basic Auth with YooKassa credentials (shopId and secretKey)
    - Headers: `Idempotence-Key` set to generated UUID, `Content-Type: application/json`
    - Body: JSON with amount (price from product), currency RUB, capture true, metadata email, confirmation with redirect and return_url, description from product title, receipt items with vat_code 1
    - Connect input from "Idempotence Key Generation"
    - On Error: continueErrorOutput

14. **Create Google Sheets Node "Save Order"**
    - Operation: Append
    - Target sheet: `orders`
    - Columns: id, email, product_id, order_datetime (map from YooKassa response and product/request data)
    - Credentials: Google Sheets OAuth2
    - Retry: 2 times
    - Connect True output from "YooKassa Request"

15. **Create Respond to Webhook Node "Respond Payment"**
    - Response Code: 200
    - Respond With: JSON
    - Response Body: `{ confirmation_url: $(YooKassa Request).item.json.confirmation.confirmation_url }`
    - Connect True output from "Save Order"

16. **Create Respond to Webhook Nodes for Errors**
    - For validation failures, missing product, YooKassa errors, and save order errors, create separate nodes returning 400 or 404 with appropriate messages
    - Connect False or error outputs accordingly:
      - "Check request values" False -> Handle Error
      - "Email validation" False -> Handle Error
      - "Check product_id" False -> Handle Error (product_id)
      - "YooKassa Request" error -> Handle Error (yookassa)
      - "Save Order" error -> Handle Error (save order)

17. **Create Webhook Node "Handle YooKassa webhook"**
    - HTTP Method: POST
    - Path: `/yoomoney`
    - Raw Body enabled
    - Response Mode: Response Node

18. **Create Switch Node "Handle Events"**
    - Routes on `$json.body.event` equals `payment.succeeded` or `refund.succeeded`
    - Connect input from "Handle YooKassa webhook"

19. **For `payment.succeeded` branch:**

    a. **Create If Node "Validation metadata"**
       - Check existence of `$json.body.object.id` and `$json.body.object.metadata.email`

    b. **Create Google Sheets Node "Get Order"**
       - Filter `orders` sheet by `id` equals `$json.body.object.id`

    c. **Create If Node "Check Order"**
       - Check if order found

    d. **Create Google Sheets Node "Save Transaction"**
       - Append to `transactions` sheet with id, email, product_id, status "succeeded", datetime captured_at
       - Connect True output from "Check Order"

    e. **Create Respond to Webhook Node "Respond payment event"**
       - Respond 200 no content
       - Connect True output from "Save Transaction"

    f. **Create Respond to Webhook Node "Handle Error webhook"**
       - Respond 200 no content for any error in this branch
       - Connect False outputs or errors from above nodes

20. **For `refund.succeeded` branch:**

    a. **Create If Node "Validation metadata_"**
       - Check existence of `$json.body.object.id`

    b. **Create Google Sheets Node "Get transaction"**
       - Filter `transactions` sheet by `id` equals `$json.body.object.id`

    c. **Create If Node "Check if transaction exists"**
       - Check if transaction found

    d. **Create Google Sheets Node "Update refund status"**
       - Update `transaction_status` to "refund" for matching transaction id

    e. **Create Respond to Webhook Node "Respond refund event"**
       - Respond 200 no content
       - Connect True output from "Update refund status"

    f. **Connect False/error outputs to "Handle Error webhook"**

21. **Create Webhook Node "Status"**
    - HTTP Method: GET
    - Path: `/status/:id`
    - Response Mode: Response Node

22. **Create If Node "Check request values_1"**
    - Validate existence of URL parameter `id`

23. **Create HTTP Request Node "YooKassa Request Status"**
    - GET `https://api.yookassa.ru/v3/payments/{{ $json.params.id }}`
    - HTTP Basic Auth with YooKassa credentials

24. **Create Respond to Webhook Node "Respond status"**
    - Return JSON with payment `id`, `status`, and `description`

25. **Create Respond to Webhook Nodes for Errors**
    - "Handle Error status" for failed YooKassa API calls
    - "Handle Error (check request)" for missing `id`

26. **Add Sticky Notes for each section**
    - "Get Products", "Payment", "Webhook", "Status", "Accept YooKassa payments and log transactions in Google Sheets" (full documentation note)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                     | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Includes full no-code payment flow integration with YooKassa and Google Sheets for products, orders, transactions. Supports payment initiation, webhook event handling, refunds, and status checking.                                            | Workflow description and sticky notes                                                                       |
| Credentials required: Google Sheets OAuth2 account with access to relevant sheets; YooKassa API credentials (shopId and secretKey) configured with HTTP Basic Auth.                                                                              | Setup instructions inside the workflow description                                                         |
| Google Sheets document must contain three sheets named `products` (with columns: product_id, title, price), `orders` (to log purchases), and `transactions` (to log payment/refund events).                                                      | Workflow configuration details                                                                              |
| The UUID generation code node provides idempotence to avoid duplicate payment creation on retries.                                                                                                                                             | Code node "Idempotence Key Generation"                                                                     |
| Email validation uses regex to ensure proper format before proceeding with payment.                                                                                                                                                             | "Email validation" node                                                                                      |
| Webhook URLs `/yoomoney` and `/payment` must be publicly accessible for YooKassa callbacks and client requests.                                                                                                                                 | Webhook nodes configuration                                                                                 |
| API usage examples and customization notes are included in the large Sticky Note2 with detailed documentation and use cases.                                                                                                                  | Sticky Note2 content                                                                                        |
| Potential improvements: add delivery notifications, replace Google Sheets with a database, integrate messaging platforms, add promo or subscription handling.                                                                                   | Sticky Note2 and workflow description                                                                       |
| Workflow version: Uses n8n nodes version 4.x features, including advanced Google Sheets and HTTP Request nodes with error continuation options.                                                                                                | Workflow metadata                                                                                           |

---

**Disclaimer:**  
The provided content is extracted from an n8n workflow automation respecting all applicable content policies. It contains no illegal or protected data and is intended solely for legal and public use.