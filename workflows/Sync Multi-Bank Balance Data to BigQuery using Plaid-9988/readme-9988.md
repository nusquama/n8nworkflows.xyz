Sync Multi-Bank Balance Data to BigQuery using Plaid

https://n8nworkflows.xyz/workflows/sync-multi-bank-balance-data-to-bigquery-using-plaid-9988


# Sync Multi-Bank Balance Data to BigQuery using Plaid

### 1. Workflow Overview

This workflow automates the process of synchronizing bank account balance data from multiple financial institutions into Google BigQuery for consolidated financial reporting and analytics. It is designed primarily for finance teams, accountants, and data engineers who manage multi-bank data aggregation and reporting using Plaid’s API and Google BigQuery.

**Use Cases:**  
- Weekly automated retrieval of bank balances from RBC, Amex, Wise, and PayPal via Plaid  
- Mapping raw account data to standardized QuickBooks Online (QBO) account names  
- Structuring and cleaning the data for downstream analytics  
- Loading the transformed data into a BigQuery table for reporting and integration with other BI tools

**Logical Blocks:**  
- **1.1 Schedule Trigger:** Initiates the workflow on a weekly schedule.  
- **1.2 Multi-Bank Balance Retrieval:** Parallel HTTP requests to Plaid API to fetch balances from RBC, Amex, Wise, and PayPal.  
- **1.3 Account Splitting:** Splits the array of accounts from each bank response into individual account items.  
- **1.4 Account Mapping:** Maps each account’s mask or identifier to a corresponding QBO account name using JavaScript code nodes for each bank.  
- **1.5 Account Merging:** Combines all mapped accounts from different banks into a single dataset.  
- **1.6 Data Structuring:** Generates UUIDs, normalizes balance fields, timestamps records, and prepares structured account records.  
- **1.7 BigQuery Preparation and Loading:** Formats the structured data into an SQL INSERT statement and executes it against Google BigQuery.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow execution once every week on Monday, establishing the periodic cadence for data synchronization.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Role:** Time-based activation of the workflow  
  - **Configuration:** Runs weekly on Monday (field: weeks, triggerAtDay: 1)  
  - **Input/Output:** No input; outputs trigger event to four HTTP request nodes for fetching balances  
  - **Edge Cases:** Misconfiguration of schedule may cause missed runs; ensure server timezone aligns with expected schedule.

#### 1.2 Multi-Bank Balance Retrieval

- **Overview:**  
  Performs four parallel HTTP POST requests to Plaid’s `/accounts/balance/get` endpoint, each using a distinct bank’s access token to retrieve current account balances.

- **Nodes Involved:**  
  - Fetch RBC Balances (Plaid)  
  - Fetch Amex Balances (Plaid)  
  - Fetch Wise Balances (Plaid)  
  - Fetch PayPal Balances (Plaid)

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** API call to Plaid to fetch account balances  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://production.plaid.com/accounts/balance/get`  
    - Body parameters include `client_id`, `secret`, and respective `access_token` (dynamic via expression)  
    - `client_id` and `secret` must be replaced with actual credentials or environment variables  
  - **Input/Output:** Input from Schedule Trigger; outputs JSON response with accounts array  
  - **Edge Cases:**  
    - API authentication failure (invalid `client_id`/`secret`/`access_token`)  
    - Network timeouts or Plaid service outages  
    - Rate limiting by Plaid API  
  - **Notes:** Each node uses a separate access token for its respective bank account.

#### 1.3 Account Splitting

- **Overview:**  
  Splits the JSON response arrays of accounts from each bank into individual items for granular processing.

- **Nodes Involved:**  
  - Split RBC Accounts  
  - Split Amex Accounts  
  - Split Wise Accounts  
  - Split PayPal Accounts

- **Node Details:**  
  - **Type:** Split Out  
  - **Role:** Normalize array responses into single account item streams  
  - **Configuration:** Field to split out is `accounts` from the HTTP response  
  - **Input/Output:** Input from respective Fetch Balances nodes; outputs individual account JSON objects  
  - **Edge Cases:**  
    - Empty or missing `accounts` field in response  
    - Malformed JSON responses

#### 1.4 Account Mapping

- **Overview:**  
  Maps each account’s mask (last digits or identifier) or official name to a predefined QuickBooks Online account name for standardized reporting.

- **Nodes Involved:**  
  - Map RBC Accounts to QBO  
  - Map Amex Accounts to QBO  
  - Map Wise Accounts to QBO  
  - Map PayPal Accounts to QBO

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:** Assigns a `qbo_account_name` to each account based on mask or fallback logic  
  - **Configuration:**  
    - Uses hardcoded mapping dictionaries per bank (e.g., RBC masks like "2245" → "RBC Business Profits-2245")  
    - If mask not found, falls back to prefixing mask or using official name  
  - **Key Expressions:** Accesses `item.json.mask` and `item.json.official_name`  
  - **Input/Output:** Input from respective Split Accounts node; outputs updated account JSON with `qbo_account_name` field  
  - **Edge Cases:**  
    - Missing or unexpected mask values  
    - Official names with inconsistent formatting  
    - The PayPal mapping node uses a mapping originally for Wise, which may be a logic oversight or intentional fallback  

#### 1.5 Account Merging

- **Overview:**  
  Combines the individual streams of mapped accounts from all banks into a single unified dataset for subsequent processing.

- **Nodes Involved:**  
  - Combine All Accounts

- **Node Details:**  
  - **Type:** Merge  
  - **Role:** Merge multiple input streams into one output  
  - **Configuration:** Set to accept 4 inputs (RBC, Amex, Wise, PayPal)  
  - **Input/Output:** Inputs from all Map Accounts nodes; outputs combined list  
  - **Edge Cases:**  
    - Delayed or missing inputs from parallel branches may cause incomplete merges  
    - Data type or structure inconsistencies between inputs

#### 1.6 Data Structuring

- **Overview:**  
  Processes the merged accounts by generating UUIDs, normalizing currency balances, adding timestamps, and constructing a structured record for database insertion.

- **Nodes Involved:**  
  - Structure Account Records

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:**  
    - Generates UUIDs from a seed composed of `account_id` and `qbo_account_name`  
    - Separates balances into USD and CAD fields based on currency code  
    - Adds metadata fields like creation dates and domain (“Plaid”)  
  - **Key Expressions:** Uses date/time ISO strings, array iteration over all input items  
  - **Input/Output:** Input from Combine All Accounts; outputs fully structured account records  
  - **Edge Cases:**  
    - Missing or null balance fields  
    - Non-USD/CAD currencies are not explicitly handled (resulting in zero balances)  
    - UUID generation relies on string hashing with some randomness, which may produce collisions in rare cases

#### 1.7 BigQuery Preparation and Loading

- **Overview:**  
  Converts structured account records into a sanitized SQL INSERT statement and loads the data into a Google BigQuery table.

- **Nodes Involved:**  
  - Prepare BigQuery Insert  
  - Load Accounts into BigQuery

- **Node Details:**  
  - **Prepare BigQuery Insert:**  
    - **Type:** Code (JavaScript)  
    - **Role:** Sanitizes strings by removing special characters and escapes backslashes, formats dates, and constructs SQL values clause for bulk insert  
    - **Input/Output:** Input from Structure Account Records; output is a JSON object with `valuesString` containing the SQL VALUES segment  
    - **Edge Cases:** Handling of nulls, special characters, and potential SQL injection mitigations  
  - **Load Accounts into BigQuery:**  
    - **Type:** Google BigQuery  
    - **Role:** Executes an SQL INSERT query into the `quickbooks.accounts` table using the formatted VALUES string  
    - **Configuration:**  
      - SQL query parameterized with `{{ $json.valuesString }}`  
      - Project ID: `n8n-self-host-461314` (replace with user project)  
    - **Input/Output:** Input from Prepare BigQuery Insert; outputs query execution response  
    - **Edge Cases:**  
      - BigQuery authentication errors (credential misconfiguration)  
      - SQL syntax errors due to malformed input strings  
      - Data type mismatches or schema conflicts in BigQuery table

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                          | Input Node(s)                                | Output Node(s)                           | Sticky Note                                                                                   |
|-----------------------------|---------------------|----------------------------------------|----------------------------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger    | Initiates workflow weekly               | None                                         | Fetch RBC Balances (Plaid), Fetch Amex Balances (Plaid), Fetch Wise Balances (Plaid), Fetch PayPal Balances (Plaid) | ## Automated Multi-Bank Balance Sync to BigQuery... (full note content in Sticky Note1)      |
| Fetch RBC Balances (Plaid) | HTTP Request        | Fetch RBC account balances via Plaid   | Schedule Trigger                             | Split RBC Accounts                     |                                                                                              |
| Fetch Amex Balances (Plaid)| HTTP Request        | Fetch Amex account balances via Plaid  | Schedule Trigger                             | Split Amex Accounts                    |                                                                                              |
| Fetch Wise Balances (Plaid)| HTTP Request        | Fetch Wise account balances via Plaid  | Schedule Trigger                             | Split Wise Accounts                    |                                                                                              |
| Fetch PayPal Balances (Plaid)| HTTP Request      | Fetch PayPal account balances via Plaid| Schedule Trigger                             | Split PayPal Accounts                  |                                                                                              |
| Split RBC Accounts         | Split Out           | Split RBC accounts array into items    | Fetch RBC Balances (Plaid)                   | Map RBC Accounts to QBO                |                                                                                              |
| Split Amex Accounts        | Split Out           | Split Amex accounts array into items   | Fetch Amex Balances (Plaid)                  | Map Amex Accounts to QBO               |                                                                                              |
| Split Wise Accounts        | Split Out           | Split Wise accounts array into items   | Fetch Wise Balances (Plaid)                  | Map Wise Accounts to QBO               |                                                                                              |
| Split PayPal Accounts      | Split Out           | Split PayPal accounts array into items | Fetch PayPal Balances (Plaid)                | Map PayPal Accounts to QBO             |                                                                                              |
| Map RBC Accounts to QBO    | Code                | Map RBC account masks to QBO names     | Split RBC Accounts                           | Combine All Accounts                   |                                                                                              |
| Map Amex Accounts to QBO   | Code                | Map Amex account masks to QBO names    | Split Amex Accounts                          | Combine All Accounts                   |                                                                                              |
| Map Wise Accounts to QBO   | Code                | Map Wise account masks to QBO names    | Split Wise Accounts                          | Combine All Accounts                   |                                                                                              |
| Map PayPal Accounts to QBO | Code                | Map PayPal account masks to QBO names  | Split PayPal Accounts                        | Combine All Accounts                   |                                                                                              |
| Combine All Accounts       | Merge               | Merge all mapped accounts into one list| Map RBC Accounts to QBO, Map Amex Accounts to QBO, Map Wise Accounts to QBO, Map PayPal Accounts to QBO | Structure Account Records              |                                                                                              |
| Structure Account Records  | Code                | Generate UUIDs, normalize and structure account data | Combine All Accounts                         | Prepare BigQuery Insert                |                                                                                              |
| Prepare BigQuery Insert    | Code                | Sanitize and format data as SQL insert | Structure Account Records                    | Load Accounts into BigQuery            |                                                                                              |
| Load Accounts into BigQuery| Google BigQuery     | Load sanitized account data into BigQuery table | Prepare BigQuery Insert                      | None                                  |                                                                                              |
| Sticky Note1               | Sticky Note         | Descriptive notes about the workflow   | None                                         | None                                  | ## Automated Multi-Bank Balance Sync to BigQuery... (full workflow explanation and setup)   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Configuration: Set to run weekly on Monday (interval: weeks, triggerAtDay: 1)

2. **Create Four HTTP Request Nodes for Balance Fetching:**  
   For each bank (RBC, Amex, Wise, PayPal):  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://production.plaid.com/accounts/balance/get`  
   - Body Parameters:  
     - `client_id`: your Plaid client ID (set as environment variable or credential)  
     - `secret`: your Plaid secret (set as environment variable or credential)  
     - `access_token`: the bank-specific access token (e.g., `=AMEX_ACCESS_TOKEN`) configured as expression or environment variable  
   - Connect Schedule Trigger output to all four HTTP Request nodes in parallel

3. **Create Split Out Nodes for Each Bank Response:**  
   For each HTTP Request node:  
   - Type: Split Out  
   - Field to split out: `accounts`  
   - Connect the corresponding HTTP Request node output to this Split Out node

4. **Create Code Nodes for Account Mapping:**  
   For each bank (RBC, Amex, Wise, PayPal):  
   - Type: Code  
   - Paste the provided JavaScript code mapping masks to QBO account names (see Block 1.4)  
   - Connect the corresponding Split Out node to this Code node

5. **Create a Merge Node:**  
   - Type: Merge  
   - Number of inputs: 4  
   - Connect all four Map Accounts code nodes to the Merge node inputs

6. **Create Structure Account Records Code Node:**  
   - Type: Code  
   - Paste the provided JavaScript code that generates UUIDs, normalizes balances, and structures the account data (see Block 1.6)  
   - Connect the Merge node output to this node

7. **Create Prepare BigQuery Insert Code Node:**  
   - Type: Code  
   - Paste the provided JavaScript code that sanitizes strings and constructs the SQL insert statement (see Block 1.7)  
   - Connect the Structure Account Records node output to this node

8. **Create Google BigQuery Node:**  
   - Type: Google BigQuery  
   - Configure credentials with OAuth2 or service account to access your BigQuery project  
   - Set Project ID (e.g., `n8n-self-host-461314`)  
   - SQL Query:  
     ```
     INSERT INTO `quickbooks.accounts`
     (
       id,
       account_id,
       date_created_at,
       name,
       active,
       classification,
       account_type,
       current_balance_usd,
       current_balance_cad,
       currency_origin,
       domain,
       account_create_time,
       account_last_update_time,
       available_balance_usd,
       available_balance_cad,
       description
     )
     VALUES
     {{ $json.valuesString }};
     ```
   - Connect the Prepare BigQuery Insert node output to this node

9. **Add Sticky Note Node (Optional):**  
   - Add a sticky note with the workflow overview, usage instructions, and setup notes for documentation

10. **Test the workflow:**  
   - Confirm all credentials are properly configured  
   - Replace placeholders (`CLIENT_ID`, `SECRET`, `ACCESS_TOKEN`s) with actual values or environment variables  
   - Run manually or wait for scheduled trigger to test full data flow

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow automatically fetches balances from multiple financial institutions using Plaid, maps them to QuickBooks account names, and loads structured records into Google BigQuery for analytics. It is intended for finance teams, accountants, and data engineers managing consolidated bank reporting in BigQuery. Setup requires adding Plaid and Google BigQuery credentials, replacing client IDs and secrets with variables, testing each connection, and scheduling the trigger as per reporting cadence. | Workflow description and setup instructions (Sticky Note1)   |

---

**Disclaimer:** The content provided is exclusively derived from an automated workflow created with n8n, respecting all current content policies with no illegal, offensive, or protected elements. All processed data is legal and public.