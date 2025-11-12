Send WooCommerce Category Alerts via WhatsApp with Rapiwa API & Google Sheets

https://n8nworkflows.xyz/workflows/send-woocommerce-category-alerts-via-whatsapp-with-rapiwa-api---google-sheets-9002


# Send WooCommerce Category Alerts via WhatsApp with Rapiwa API & Google Sheets

### 1. Workflow Overview

This workflow automates sending WhatsApp alerts to WooCommerce customers whenever a new product category is created in the store. It listens to WooCommerce category creation events via a webhook, fetches customer data, cleans and verifies their phone numbers using the Rapiwa API, sends promotional WhatsApp messages only to verified numbers, and logs both verified and unverified contacts into Google Sheets.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and Category Data Formatting:** Receives new category data from WooCommerce via webhook and formats it.
- **1.2 Customer Retrieval and Batching:** Fetches WooCommerce customers, limits the number processed, and breaks them into batches.
- **1.3 Customer Data Cleaning:** Cleans and formats customer phone numbers, names, emails, and addresses.
- **1.4 WhatsApp Number Verification:** Validates cleaned phone numbers with Rapiwa API to confirm WhatsApp availability.
- **1.5 Conditional Branching:** Routes customers into verified vs unverified based on WhatsApp number verification.
- **1.6 WhatsApp Messaging:** Sends promotional WhatsApp messages to verified customers using Rapiwa.
- **1.7 Logging and Rate Limiting:** Logs customer data (verified/unverified) into Google Sheets and inserts a wait period before continuing processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Category Data Formatting

- **Overview:** This block receives the WooCommerce webhook payload for a newly created product category and extracts relevant category details into a structured format.
- **Nodes Involved:** `Webhook`, `Format Webhook Response Data`
  
**Nodes:**

- **Webhook**  
  - Type: Webhook node  
  - Role: Listens for HTTP POST requests from WooCommerce on category creation  
  - Configuration:  
    - HTTP Method: POST  
    - Path: Unique webhook path  
  - Inputs: External HTTP POST payload from WooCommerce  
  - Outputs: Raw webhook JSON with category info  
  - Edge Cases: Invalid payloads, unauthorized requests if webhook secret mismatches, network issues

- **Format Webhook Response Data**  
  - Type: Code node (JavaScript)  
  - Role: Parses and restructures the incoming webhook JSON to extract category fields such as id, name, slug, description, parent category, display type, and thumbnail ID  
  - Key Expression: Extracts data from `items[0].json.body` and returns a simplified JSON object for reuse in downstream nodes  
  - Inputs: Raw webhook output  
  - Outputs: Parsed category JSON  
  - Edge Cases: Missing or malformed category fields, null or undefined optional fields handled with optional chaining  
  - Version: Uses n8n Code node v2 style

---

#### 2.2 Customer Retrieval and Batching

- **Overview:** Fetches all WooCommerce customers via API, limits the number to 10 for manageable processing, and splits them into batches for sequential handling.
- **Nodes Involved:** `Get many customers`, `Limit`, `Loop Over Items`
  
**Nodes:**

- **Get many customers**  
  - Type: WooCommerce node  
  - Role: Retrieves all customer records from WooCommerce store via REST API  
  - Configuration:  
    - Resource: Customer  
    - Operation: Get All  
    - Return All: true (fetches all customers)  
  - Credentials: WooCommerce API credentials (OAuth or API key)  
  - Inputs: Category formatted data (triggered after category extraction)  
  - Outputs: List of all customers  
  - Edge Cases: API rate limits, authentication failures, empty customer list

- **Limit**  
  - Type: Limit node  
  - Role: Restricts the number of customer records passed downstream to 10, for testing or batch processing control  
  - Configuration:  
    - Max Items: 10  
  - Inputs: Customer list  
  - Outputs: Maximum 10 customers  
  - Edge Cases: If fewer than 10 customers, passes all; no failure expected

- **Loop Over Items**  
  - Type: SplitInBatches node  
  - Role: Splits the limited customer list into individual items for processing one at a time  
  - Configuration: Default batch size 1  
  - Inputs: Limited customers  
  - Outputs: Single customer per batch  
  - Edge Cases: Empty input leads to no batches; batch processing errors possible if downstream nodes fail

---

#### 2.3 Customer Data Cleaning

- **Overview:** Cleans and prepares customer contact data by extracting and formatting phone numbers, emails, names, and addresses from billing info.
- **Nodes Involved:** `Clean Number`
  
**Node:**

- **Clean Number**  
  - Type: Code node (JavaScript)  
  - Role:  
    - Extracts customer billing phone number and strips all non-numeric characters to standardize the number.  
    - Extracts email from billing or customer data.  
    - Concatenates billing address fields into a single string.  
    - Formats customer's full name from first and last name.  
  - Key Expression: Uses regex to remove non-digits from phone, joins address parts, trims names  
  - Inputs: Single customer item from Loop Over Items  
  - Outputs: JSON with cleaned phone number (`number`), email, full address, and name  
  - Edge Cases: Missing phone or email fields, empty address components, non-string phone numbers handled gracefully

---

#### 2.4 WhatsApp Number Verification

- **Overview:** Validates cleaned phone numbers by checking if they are registered and active WhatsApp numbers via Rapiwa API.
- **Nodes Involved:** `Check valid whatsapp number Using Rapiwa`
  
**Node:**

- **Check valid whatsapp number Using Rapiwa**  
  - Type: HTTP Request node  
  - Role: Sends POST request to Rapiwa API endpoint `/api/verify-whatsapp` with the cleaned phone number  
  - Configuration:  
    - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
    - Method: POST  
    - Authentication: HTTP Bearer token (Rapiwa Bearer Auth credentials)  
    - Body parameter: `{ number: cleaned phone number }`  
  - Inputs: Output from Clean Number node  
  - Outputs: API JSON response including `data.exists` boolean indicating WhatsApp number validity  
  - Edge Cases: API failures (timeouts, 401 unauthorized if token invalid), malformed phone number errors, rate limits

---

#### 2.5 Conditional Branching

- **Overview:** Determines the path for each customer based on WhatsApp verification result — verified numbers proceed to messaging, unverified numbers skip messaging but are logged.
- **Nodes Involved:** `If`
  
**Node:**

- **If**  
  - Type: If node  
  - Role: Checks if `data.exists` in the HTTP response is `true` (valid WhatsApp number)  
  - Configuration:  
    - Condition: Boolean equals true for `={{ $json.data.exists }}`  
  - Inputs: Output from WhatsApp verification node  
  - Outputs:  
    - True branch: for verified numbers  
    - False branch: for unverified numbers  
  - Edge Cases: Missing `data.exists` key, unexpected data types, expression evaluation errors

---

#### 2.6 WhatsApp Messaging

- **Overview:** Sends a promotional WhatsApp message to verified customers using Rapiwa's send-message API.
- **Nodes Involved:** `Send Message Using Rapiwa`
  
**Node:**

- **Send Message Using Rapiwa**  
  - Type: HTTP Request node  
  - Role: Sends POST request to Rapiwa API endpoint `/api/send-message` to deliver a text message  
  - Configuration:  
    - URL: `https://app.rapiwa.com/api/send-message`  
    - Method: POST  
    - Authentication: HTTP Bearer token (Rapiwa Bearer Auth)  
    - Body parameters:  
      - `number`: verified WhatsApp number from verification response  
      - `message_type`: `"text"`  
      - `message`: Personalized message using customer name and new category info from workflow context, e.g.:  
        ```
        Hey [Name],
        We just launched a new category: *[Category Name]*!
        Explore now👉 https://rahim.spagreen.info/product-category/[slug]
        Don’t miss it!
        Cheers,
        Team SpaGreen Creative
        ```
  - Inputs: Verified WhatsApp numbers and category data  
  - Outputs: API response of message send status  
  - Edge Cases: API failures, invalid message formats, rate limiting, message delivery errors

---

#### 2.7 Logging and Rate Limiting

- **Overview:** Logs customer contact info into Google Sheets with status `verified` or `unverified` based on verification. Adds wait time to avoid API throttling and controls loop pacing.
- **Nodes Involved:** `verified append row in sheet`, `unverified append row in sheet`, `Wait`
  
**Nodes:**

- **verified append row in sheet**  
  - Type: Google Sheets node  
  - Role: Appends a row to a Google Sheet with customer details and status set to `verified`  
  - Configuration:  
    - Operation: Append  
    - Document ID: Google Sheet ID with customer data  
    - Sheet Name: Target sheet tab (gid=0)  
    - Columns mapped: name (note trailing space), number, email, address, category name (`catagoris`), description, status=`verified`  
  - Credentials: Google Sheets OAuth2 credentials  
  - Inputs: Data from Send Message Using Rapiwa and cleaned customer info  
  - Edge Cases: Sheet access issues, quota limits, permission errors

- **unverified append row in sheet**  
  - Type: Google Sheets node  
  - Role: Appends a row to the same Google Sheet but with status `unverified` for customers whose WhatsApp number is invalid  
  - Configuration: Same as verified append node, except number is from cleaned data and status set to `unverified`  
  - Credentials: Google Sheets OAuth2 credentials  
  - Inputs: Data from If node false branch and cleaned customer info  
  - Edge Cases: Same as verified append node

- **Wait**  
  - Type: Wait node  
  - Role: Introduces a delay between processing batches/customers to avoid hitting API rate limits or Google Sheets quotas  
  - Configuration: Default wait time (unspecified, configurable)  
  - Inputs: Output from both append row nodes  
  - Outputs: Loops back to `Loop Over Items` to process next customer  
  - Edge Cases: Workflow stuck if wait time too long, possible workflow timeouts if wait too long

---

### 3. Summary Table

| Node Name                         | Node Type              | Functional Role                                           | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                                         |
|----------------------------------|------------------------|-----------------------------------------------------------|----------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                          | Webhook                | Receives WooCommerce category creation webhook POST       | External HTTP POST                | Format Webhook Response Data      | ## Webhook<br>Listens for incoming HTTP POST requests — product category creation data from WooCommerce.                            |
| Format Webhook Response Data     | Code                   | Parses and formats category data from webhook              | Webhook                         | Get many customers                | ## & Format Webhook Response Data<br>Parses and reformats incoming category data into structured JSON.                             |
| Get many customers               | WooCommerce            | Fetches all WooCommerce customers                           | Format Webhook Response Data      | Limit                           | ## Get many customers<br>Fetches all customer data from WooCommerce API.                                                           |
| Limit                           | Limit                  | Limits number of customers processed (max 10)              | Get many customers                | Loop Over Items                  | ## Limit<br>Limits customers to 10 for testing or batch control.<br>## & Loop Over Items<br>Iterates over customers one at a time.  |
| Loop Over Items                 | SplitInBatches          | Processes customers one at a time in batches               | Limit                           | Clean Number                    | ## Limit<br>Limits customers to 10 for testing or batch control.<br>## & Loop Over Items<br>Iterates over customers one at a time.  |
| Clean Number                   | Code                   | Cleans and formats customer phone, email, name, address   | Loop Over Items                  | Check valid whatsapp number Using Rapiwa | ## Clean Number<br>Removes non-numeric chars from phone, formats name/email/address.                                                |
| Check valid whatsapp number Using Rapiwa | HTTP Request           | Verifies if phone number is a valid WhatsApp number via API | Clean Number                    | If                             | ## Check valid WhatsApp number Using Rapiwa<br>Verifies WhatsApp number via Rapiwa API.                                             |
| If                             | If                     | Routes based on WhatsApp number validity                    | Check valid whatsapp number Using Rapiwa | Send Message Using Rapiwa (true), unverified append row in sheet (false) | ## If<br>Checks if WhatsApp number exists to branch workflow.                                                                       |
| Send Message Using Rapiwa       | HTTP Request           | Sends promotional WhatsApp message via Rapiwa API          | If (true)                      | verified append row in sheet     | ## Send Message Using Rapiwa<br>Sends WhatsApp message to verified numbers.                                                        |
| verified append row in sheet    | Google Sheets          | Logs verified customer data to Google Sheets               | Send Message Using Rapiwa        | Wait                           | ## verified append row in sheet<br>Appends verified user data to Google Sheets.                                                    |
| unverified append row in sheet  | Google Sheets          | Logs unverified customer data to Google Sheets             | If (false)                     | Wait                           | ## unverified append row in sheet<br>Appends unverified user data to Google Sheets.                                                |
| Wait                           | Wait                   | Adds delay between processing customers to avoid rate limits | verified append row in sheet, unverified append row in sheet | Loop Over Items                | ## Wait<br>Introduces delay to avoid API or Sheets rate limits before next iteration.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique (e.g., `1c217a0f-503f-49a4-b0c5-1aa431616177`)  
   - Purpose: Receive WooCommerce category creation payload

2. **Create Code Node: Format Webhook Response Data**  
   - Input: Webhook output  
   - Code: Extract `body` from webhook JSON and return fields: id, name, slug, description, parent, display_type, thumbnail_id  
   - Output: Structured category JSON

3. **Create WooCommerce Node: Get many customers**  
   - Resource: Customer  
   - Operation: Get All  
   - Return All: true  
   - Credentials: Configure WooCommerce API credentials (API key/secret or OAuth)  
   - Connect input from `Format Webhook Response Data`

4. **Create Limit Node**  
   - Max Items: 10 (adjustable)  
   - Connect input from `Get many customers`

5. **Create SplitInBatches Node: Loop Over Items**  
   - Batch Size: 1 (default)  
   - Connect input from `Limit`

6. **Create Code Node: Clean Number**  
   - Input: Single customer item from `Loop Over Items`  
   - Code:  
     - Extract `billing.phone`, remove non-digits  
     - Extract email from billing or customer data  
     - Concatenate billing address fields into one string  
     - Format full name from billing first and last names  
     - Return JSON with keys: number, email, address, name  
   - Connect input from `Loop Over Items`

7. **Create HTTP Request Node: Check valid whatsapp number Using Rapiwa**  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Method: POST  
   - Authentication: HTTP Bearer token (create Rapiwa Bearer Auth credentials with your token)  
   - Body Parameters: `number` = `{{$json.number}}`  
   - Connect input from `Clean Number`

8. **Create If Node**  
   - Condition: Boolean expression `{{$json.data.exists}} === true`  
   - Connect input from `Check valid whatsapp number Using Rapiwa`  
   - True branch: verified  
   - False branch: unverified

9. **Create HTTP Request Node: Send Message Using Rapiwa**  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Method: POST  
   - Authentication: HTTP Bearer token (same Rapiwa credentials)  
   - Body Parameters:  
     - `number`: `{{$json.data.number}}` from verification response  
     - `message_type`: `"text"`  
     - `message`: Use expression to personalize message incorporating `$('Clean Number').item.json.name` and `$('Format Webhook Response Data').item.json.name` and slug for category URL  
   - Connect input from If node true branch

10. **Create Google Sheets Node: verified append row in sheet**  
    - Operation: Append  
    - Document ID: Google Sheets document ID (your sheet)  
    - Sheet Name: e.g., `gid=0` or sheet tab name  
    - Columns: map `name ` (with trailing space), `number`, `email`, `address`, `catagoris` (category name), `description`, `status` = `verified`  
    - Credentials: Google Sheets OAuth2  
    - Connect input from `Send Message Using Rapiwa`

11. **Create Google Sheets Node: unverified append row in sheet**  
    - Same as verified append node, but `status` = `unverified`  
    - Connect input from If node false branch  
    - Use cleaned number instead of verified number  

12. **Create Wait Node**  
    - Default wait time (set as needed, e.g., seconds)  
    - Connect inputs from both Google Sheets append nodes (verified and unverified)  
    - Connect output back to `Loop Over Items` node to process next customer

13. **Set Execution Order**  
    - Webhook → Format Webhook Response Data → Get many customers → Limit → Loop Over Items → Clean Number → Check valid whatsapp number Using Rapiwa → If → (True → Send Message Using Rapiwa → verified append row in sheet → Wait) and (False → unverified append row in sheet → Wait) → Loop Over Items (loop continues)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| The Google Sheet must have columns named exactly: `name ` (with trailing space), `number`, `email`, `address`, `catagoris`, `description`, `status`. Do not remove or rename columns, including trailing spaces.                      | Google Sheets schema requirement                                                                                         |
| Rapiwa API is unofficial and may not guarantee 100% delivery rates. Use opt-in lists to avoid spamming customers.                                                                                                                  | Usage advisory                                                                                                           |
| Adjust the Wait node delay to avoid Rapiwa API or Google Sheets rate limits and avoid workflow timeouts.                                                                                                                           | Rate limiting handling                                                                                                   |
| Sample Google Sheet used in the workflow is accessible here: https://docs.google.com/spreadsheets/d/1SbBOtdqaA9eUmgv2W4MXU0-TODjHix-IrmQmuapiSJA/edit?usp=sharing                                                                       | Sample sheet for reference                                                                                              |
| WooCommerce store must have REST API enabled with appropriate credentials configured in n8n.                                                                                                                                       | WooCommerce API setup                                                                                                    |
| Rapiwa Bearer Token must be stored securely in n8n credentials and linked to HTTP Request nodes requiring it.                                                                                                                     | Credential management                                                                                                    |
| The WhatsApp message template is customizable in the HTTP Request node body; use expressions to inject dynamic values.                                                                                                            | Message content customization                                                                                           |
| Workflow includes extensive logging to Google Sheets to track both verified and unverified customers for analysis and auditing.                                                                                                    | Logging and auditing                                                                                                    |
| Workflow can be extended to notify store admins on message failures or to handle different categories with varied messages by modifying code and message template nodes.                                                           | Customization ideas                                                                                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.