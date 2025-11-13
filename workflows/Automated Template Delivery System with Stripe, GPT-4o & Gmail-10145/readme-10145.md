Automated Template Delivery System with Stripe, GPT-4o & Gmail

https://n8nworkflows.xyz/workflows/automated-template-delivery-system-with-stripe--gpt-4o---gmail-10145


# Automated Template Delivery System with Stripe, GPT-4o & Gmail

### 1. Workflow Overview

This workflow automates the delivery of purchased automation templates to customers after successful payments via Stripe. It integrates Stripe payment data, Google Sheets for automation metadata and purchase tracking, Azure OpenAI GPT-4o-mini for personalized email generation, and Gmail for sending emails. The workflow runs daily, fetching new successful purchases, matching them to automation templates, generating tailored thank-you emails with access details, sending these emails, and logging transactions for audit.

Logical blocks:

- **1.1 Scheduled Trigger & Stripe Data Collection:** Daily trigger initiates fetching all Stripe charges, filtering for successful payments.
- **1.2 Payment & Product Detail Enrichment:** Expands charge data with payment intent and product metadata from Stripe.
- **1.3 Data Merging & Validation:** Combines and normalizes charge, payment intent, and product info. Validates required fields.
- **1.4 Automation Metadata Lookup:** Uses AI-assisted Google Sheets agent and direct sheet lookup to find matching automation details.
- **1.5 Purchase Deduplication & Filtering:** Checks purchase history sheet to exclude already-processed customers.
- **1.6 Customer Matching & Final Validation:** Matches charges to automations, confirms presence of data, and prepares final purchase list.
- **1.7 Email Generation & Dispatch Loop:** Iterates over new purchases, uses Azure OpenAI GPT-4o-mini to create personalized HTML emails, sends via Gmail.
- **1.8 Logging & Tracking:** Appends or updates purchase data in Google Sheets for audit and reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Stripe Data Collection

- **Overview:**  
  Automatically triggers daily at 7 AM IST to fetch all Stripe charges and filter only successful payments for processing.

- **Nodes Involved:**  
  - Schedule Trigger Daily  
  - Stripe Data Collection  
  - Filter – Successful Charges  
  - Get Payment Intent (From Charge)  
  - If1 (Check existence of order_reference)  
  - Get Product Details (from Payment Intent)  
  - Merge Charge + PaymentIntent + Product

- **Node Details:**

  - **Schedule Trigger Daily**  
    - Type: Schedule Trigger  
    - Configuration: Triggers once daily at 7 AM IST.  
    - Inputs: None (start node)  
    - Outputs: Stripe Data Collection  
    - Edge Cases: Timezone misconfiguration might cause off-schedule runs.

  - **Stripe Data Collection**  
    - Type: Stripe API (charge resource, getAll)  
    - Configuration: Retrieves all charge objects from Stripe, no limit (returnAll=true).  
    - Credentials: Stripe API (Techdome account)  
    - Inputs: From Schedule Trigger  
    - Outputs: Filter – Successful Charges  
    - Potential Failures: API rate limits, authentication errors.

  - **Filter – Successful Charges**  
    - Type: Filter Node  
    - Configuration: Filters charges where status equals "succeeded".  
    - Inputs: Stripe charges  
    - Outputs: Get Payment Intent (From Charge)  
    - Edge Cases: Charges with unexpected status values may be excluded.

  - **Get Payment Intent (From Charge)**  
    - Type: HTTP Request (Stripe API)  
    - Configuration: Fetches payment intent details using payment_intent ID from charge JSON.  
    - Credentials: Stripe API  
    - Inputs: Filtered successful charges  
    - Outputs: If1 and Merge Charge + PaymentIntent + Product (conditional)  
    - Edge Cases: Missing or invalid payment_intent ID causes failure.

  - **If1**  
    - Type: If Condition  
    - Configuration: Checks existence of order_reference inside payment intent's payment_details.  
    - Inputs: Payment intent data  
    - Outputs: Get Product Details (from Payment Intent) if order_reference exists  
    - Edge Cases: Missing order_reference causes product detail fetch to be skipped.

  - **Get Product Details (from Payment Intent)**  
    - Type: HTTP Request (Stripe API)  
    - Configuration: Fetches product details using order_reference as product ID.  
    - Credentials: Stripe API  
    - Inputs: If1 (when order_reference exists)  
    - Outputs: Merge Charge + PaymentIntent + Product  
    - Edge Cases: Invalid product ID or missing product causes empty product details.

  - **Merge Charge + PaymentIntent + Product**  
    - Type: Merge Node (3 inputs)  
    - Configuration: Combines charge, payment intent, and product data into one structured record.  
    - Inputs: Charge, Payment Intent, Product nodes  
    - Outputs: Merge Logic Format Data  
    - Edge Cases: Missing any input causes incomplete merged data.

- **Sticky Notes:**  
  - "Trigger & Data Fetch" (Schedule Trigger Daily)  
  - "Stripe Data Collection"  
  - "Filter – Successful Charges"  
  - "Get Payment Intent (from Charge)"  
  - "Get Product Details (from Payment Intent)"  
  - "Merge Charge + PaymentIntent + Product"  

---

#### 1.2 Data Merging & Validation

- **Overview:**  
  Normalizes and merges Stripe data, validates required fields to ensure only complete records proceed.

- **Nodes Involved:**  
  - Merge Logic Format Data (Code)  
  - Check Required Fields (If)

- **Node Details:**

  - **Merge Logic Format Data**  
    - Type: Code Node (JavaScript)  
    - Configuration: Separates charges, intents, products; merges into unified JSON with keys such as customer_name, email, product_name, amount, receipt_url, created date.  
    - Inputs: Merge Charge + PaymentIntent + Product  
    - Outputs: Check Required Fields  
    - Edge Cases: Missing payment intent or product causes null fields; code skips incomplete merges.

  - **Check Required Fields**  
    - Type: If Condition  
    - Configuration: Validates that order_reference, product_name, customer_name, and email are all non-empty before continuing.  
    - Inputs: Merged logic data  
    - Outputs: AI Agent → Google Sheets Lookup (if true), Combine Stripe + Sheet Data (if false)  
    - Edge Cases: Any missing required field blocks downstream processing.

- **Sticky Notes:**  
  - "Merge Logic Format Data"  
  - "Check Required Fields"  

---

#### 1.3 Automation Metadata Lookup

- **Overview:**  
  Uses AI and direct Google Sheets lookup to find matching automation template metadata by product name.

- **Nodes Involved:**  
  - AI Agent → Google Sheets Lookup (LangChain agent)  
  - Azure OpenAI Chat Model1 (GPT-4o-mini)  
  - Structured Output Parser1  
  - Get row(s) in sheet in Google Sheets

- **Node Details:**

  - **AI Agent → Google Sheets Lookup**  
    - Type: LangChain AI Agent Node  
    - Configuration: Receives product_name as input text; queries Google Sheets via embedded Sheet tool for exact matches on "Name" column; returns full matching rows or error messages in strict JSON format.  
    - Inputs: Check Required Fields (true branch)  
    - Outputs: Check Match Found  
    - Edge Cases: No matches, multiple matches, or Sheet access errors produce controlled JSON responses.

  - **Azure OpenAI Chat Model1**  
    - Type: LangChain LM Chat Azure OpenAI  
    - Configuration: GPT-4o-mini model used as underlying LLM for AI Agent → Google Sheets Lookup.  
    - Credentials: Azure OpenAI account  
    - Inputs: Part of AI Agent → Google Sheets Lookup (ai_languageModel connection)  
    - Outputs: Structured Output Parser1

  - **Structured Output Parser1**  
    - Type: LangChain Structured Output Parser  
    - Configuration: Parses AI agent output JSON with schema including row_number, Name, Google Drive Link, Password, Date fields.  
    - Inputs: Azure OpenAI Chat Model1  
    - Outputs: AI Agent → Google Sheets Lookup

  - **Get row(s) in sheet in Google Sheets**  
    - Type: Google Sheets Tool (LangChain ai_tool)  
    - Configuration: Used by AI Agent to perform the actual sheet query; accesses "n8n Automations – Zip Files" document/sheet.  
    - Credentials: Google Sheets OAuth2  
    - Inputs: AI Agent → Google Sheets Lookup (ai_tool connection)  
    - Outputs: AI Agent → Google Sheets Lookup

- **Sticky Notes:**  
  - "AI Agent → Google Sheets Lookup"  
  - "AI Agent  → Google Sheets Lookup" (AI Agent node and Google Sheets Tool)  

---

#### 1.4 Purchase Deduplication & Filtering

- **Overview:**  
  Cross-references purchase data with existing logged purchases to avoid duplicate processing.

- **Nodes Involved:**  
  - Get row(s) in sheet – Purchase Sheet Lookup (Google Sheets)  
  - Combine Stripe + Sheet Data (Merge)  
  - Customer matching (Code)  
  - SQL Combine (Merge with SQL query)

- **Node Details:**

  - **Get row(s) in sheet – Purchase Sheet Lookup**  
    - Type: Google Sheets  
    - Configuration: Reads all existing rows from "Automation purchase sheet" to obtain previously logged customers.  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Combine Stripe + Sheet Data  
    - Outputs: SQL Combine  
    - Edge Cases: Sheet access failure or empty sheet returns no prior records.

  - **Combine Stripe + Sheet Data**  
    - Type: Merge Node  
    - Configuration: Joins Stripe purchase data with automation metadata from AI Lookup into one dataset.  
    - Inputs: Check Match Found (true branch), Check Required Fields (false branch)  
    - Outputs: Customer matching and Get row(s) in sheet – Purchase Sheet Lookup  
    - Edge Cases: Mismatched or missing data leads to incomplete merges.

  - **Customer matching**  
    - Type: Code Node (JavaScript)  
    - Configuration: Matches successful Stripe charges to automation rows by product name, comparing trimmed names without "[n8n]" suffix. Filters only succeeded payments, outputs matched purchase info with customer and payment details.  
    - Inputs: Combine Stripe + Sheet Data  
    - Outputs: SQL Combine  
    - Edge Cases: No matches found returns error JSON with diagnostic info.

  - **SQL Combine**  
    - Type: Merge Node (combineBySql mode)  
    - Configuration: SQL query excludes customers already present in purchase sheet by matching emails or customer names case-insensitively.  
    - Inputs: Customer matching, Get row(s) in sheet – Purchase Sheet Lookup  
    - Outputs: Check Automation Exists  
    - Edge Cases: SQL syntax errors or empty inputs.

- **Sticky Notes:**  
  - "Get row(s) in sheet – Purchase Sheet Lookup"  
  - "Combine Stripe + Sheet Data"  
  - "Customer Matching"  
  - "SQL Combine"  

---

#### 1.5 Customer Validation & Loop Setup

- **Overview:**  
  Ensures automation data exists before processing and sets up batch iteration over new purchases.

- **Nodes Involved:**  
  - Check Automation Exists (If)  
  - Loop Over Items of New Purchases (SplitInBatches)

- **Node Details:**

  - **Check Automation Exists**  
    - Type: If Condition  
    - Configuration: Checks that automationName field exists and is non-empty, acting as a guard to prevent invalid records.  
    - Inputs: SQL Combine  
    - Outputs: Loop Over Items of New Purchases  
    - Edge Cases: Missing automationName stops processing.

  - **Loop Over Items of New Purchases**  
    - Type: SplitInBatches  
    - Configuration: Iterates over each new purchase record in batches (default batch size) to handle email generation and sending sequentially.  
    - Inputs: Check Automation Exists (true branch)  
    - Outputs: AI Agent – Email Composer (main branch), or empty (no further action)  
    - Edge Cases: Large batch sizes may cause rate limits or timeouts.

- **Sticky Notes:**  
  - "Check Automation Exists"  
  - "Loop Over Items of New Purchases"  

---

#### 1.6 Email Generation & Dispatch

- **Overview:**  
  Uses Azure OpenAI GPT-4o-mini to generate personalized, formatted HTML thank-you emails with access details, then sends these emails via Gmail.

- **Nodes Involved:**  
  - AI Agent – Email Composer (LangChain agent)  
  - Azure OpenAI Chat Model (GPT-4o-mini)  
  - Structured Output Parser  
  - Send a message (Gmail)  
  - Append or update row in sheet  For  Tracking (Google Sheets)

- **Node Details:**

  - **AI Agent – Email Composer**  
    - Type: LangChain AI Agent  
    - Configuration: Receives customer and purchase details as structured input; prompts GPT-4o-mini to create friendly, professional HTML emails with gratitude, access link, password, onboarding tip, and sign-off by Rahul Joshi.  
    - Inputs: Loop Over Items of New Purchases  
    - Outputs: Send a message (Gmail)  
    - Edge Cases: AI generation failures or malformed output.

  - **Azure OpenAI Chat Model**  
    - Type: LangChain LM Chat Azure OpenAI  
    - Configuration: GPT-4o-mini model used as engine behind email composer agent.  
    - Credentials: Azure OpenAI account  
    - Inputs: AI Agent – Email Composer (ai_languageModel)  
    - Outputs: Structured Output Parser

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Configuration: Parses generated email JSON with "Subject" and "Body" fields (HTML).  
    - Inputs: Azure OpenAI Chat Model  
    - Outputs: AI Agent – Email Composer

  - **Send a message (Gmail)**  
    - Type: Gmail Node  
    - Configuration: Sends email to customerEmail with subject and HTML body from AI output. Uses Gmail OAuth2 credentials.  
    - Inputs: AI Agent – Email Composer  
    - Outputs: Append or update row in sheet  For  Tracking  
    - Edge Cases: Gmail quota limits, authentication errors, invalid emails.

  - **Append or update row in sheet  For  Tracking**  
    - Type: Google Sheets  
    - Configuration: Logs each processed purchase with name, email, price, status, and template purchased in "Automation purchase sheet". Uses appendOrUpdate operation matching on "name" column.  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Send a message (Gmail)  
    - Outputs: Loop Over Items of New Purchases (empty to continue next batch)  
    - Edge Cases: Sheet access failures, data mismatches.

- **Sticky Notes:**  
  - "AI Agent – Email Composer"  
  - "Azure OpenAI Chat Model"  
  - "Structured Output Parser"  
  - "Send a message (Gmail)"  
  - "Append or update row in sheet  For  Tracking"  

---

#### 1.7 Auxiliary Nodes

- **Customer matching (Code Node)**  
  Matches automation rows with charges by product name, ignoring "[n8n]" suffix and trimming whitespace. Filters only succeeded payments.  
- **Merge Logic Format Data (Code Node)**  
  Normalizes and combines data from Stripe charge, payment intent, and product nodes into a single, clean JSON structure.  
- Various Sticky Notes provide explanatory comments on blocks and nodes.

---

### 3. Summary Table

| Node Name                             | Node Type                             | Functional Role                            | Input Node(s)                             | Output Node(s)                         | Sticky Note                                                                                              |
|-------------------------------------|-------------------------------------|--------------------------------------------|------------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger Daily               | Schedule Trigger                    | Triggers workflow daily at 7 AM IST         | None                                     | Stripe Data Collection                 | ## Trigger & Data Fetch 🟢 Schedule Trigger Triggers every morning (7 AM IST) to fetch latest successful Stripe purchases. |
| Stripe Data Collection              | Stripe API (charge, getAll)         | Fetches all Stripe charges                   | Schedule Trigger Daily                   | Filter – Successful Charges            | ## Stripe Data Collection 💳 Get many charges (Stripe) Fetches all charge data from Stripe; later filtered to keep only completed payments. |
| Filter – Successful Charges         | Filter                             | Filters only succeeded charges               | Stripe Data Collection                   | Get Payment Intent (From Charge)       | ## Filter – Successful Charges 🔍 Status check = succeeded Removes failed/pending charges so only paid orders continue. |
| Get Payment Intent (From Charge)    | HTTP Request (Stripe API)           | Fetches payment intent details               | Filter – Successful Charges              | If1, Merge Charge + PaymentIntent + Product | ## Get Payment Intent (from Charge) 🔗 Stripe Payment Intent lookup Expands each charge with its payment_intent details for richer context. |
| If1                                | If Condition                      | Checks for existence of order_reference      | Get Payment Intent (From Charge)         | Get Product Details (from Payment Intent) |                                                                                                        |
| Get Product Details (from Payment Intent) | HTTP Request (Stripe API)           | Fetches product details from Stripe          | If1                                     | Merge Charge + PaymentIntent + Product | ## Get Product Details (from Payment Intent) 🧾 Stripe Product fetch Uses order reference from payment_intent to get product name/description/image. |
| Merge Charge + PaymentIntent + Product | Merge                             | Combines charge, payment intent, product data | Get Payment Intent (From Charge), If1, Get Product Details (from Payment Intent) | Merge Logic Format Data                | ## Merge Charge + PaymentIntent + Product 🔄 Data merge hub Combines charge + payment_intent + product into one unified record per purchase. |
| Merge Logic Format Data             | Code                              | Normalizes and structures merged data        | Merge Charge + PaymentIntent + Product  | Check Required Fields                  | ## Merge Logic Format Data 🧠 Normalize fields Builds clean JSON (customer_name, email, product_name, amount, currency, status, receipt_url, created). |
| Check Required Fields               | If Condition                      | Validates essential fields presence           | Merge Logic Format Data                  | AI Agent → Google Sheets Lookup, Combine Stripe + Sheet Data | ## Check Required Fields 🚦 Field validation Ensures order_reference, product_name, customer_name, and email are present before moving on. |
| AI Agent → Google Sheets Lookup    | LangChain AI Agent                | Searches Google Sheet for automation metadata | Check Required Fields (true branch)     | Check Match Found                     | ## AI Agent → Google Sheets Lookup 🔍 Automation metadata lookup Finds matching automation by product/Name in “n8n Automations – Zip Files” sheet. |
| Azure OpenAI Chat Model1            | LangChain LM Chat Azure OpenAI    | GPT-4o-mini model for AI Agent → Google Sheets Lookup | AI Agent → Google Sheets Lookup (ai_languageModel) | Structured Output Parser1              |                                                                                                        |
| Structured Output Parser1           | LangChain Structured Output Parser | Parses AI Agent → Google Sheets Lookup output | Azure OpenAI Chat Model1                 | AI Agent → Google Sheets Lookup       |                                                                                                        |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool (LangChain)    | Performs actual sheet query for AI Agent      | AI Agent → Google Sheets Lookup (ai_tool) | AI Agent → Google Sheets Lookup       |                                                                                                        |
| Combine Stripe + Sheet Data         | Merge                             | Joins Stripe purchase data with automation metadata | Check Match Found (true branch), Check Required Fields (false branch) | Customer matching, Get row(s) in sheet – Purchase Sheet Lookup | ##  Combine Stripe + Sheet Data 🔄 Stripe × Sheet join Brings together payment info and automation metadata for a complete record. |
| Get row(s) in sheet – Purchase Sheet Lookup | Google Sheets                    | Reads existing purchase log for deduplication | Combine Stripe + Sheet Data              | SQL Combine                          | ## Get row(s) in sheet – Purchase Sheet Lookup 🧾 De-dup check Reads existing entries in “Automation Purchase Sheet” to avoid logging duplicates. |
| Customer matching                  | Code                              | Matches charges to automation rows by product name | Combine Stripe + Sheet Data              | SQL Combine                          | ## Customer Matching 🧠 Product-name matcher Matches successful Stripe charges to automation rows (handles “[n8n]” suffix and trims). Outputs final, ready-to-email JSON. |
| SQL Combine                      | Merge (combineBySql)                | Excludes already-logged customers by SQL join | Customer matching, Get row(s) in sheet – Purchase Sheet Lookup | Check Automation Exists               | ## SQL Combine 🧮 Exclude already-logged customers SQL logic filters out rows already present in the purchase sheet. |
| Check Automation Exists           | If Condition                      | Verifies automation data presence before proceeding | SQL Combine                             | Loop Over Items of New Purchases     | ##  Check Automation Exists ⚡ Final guard Verifies automation data is present before generating and sending emails. |
| Loop Over Items of New Purchases  | SplitInBatches                   | Iterates over new purchases for email processing | Check Automation Exists                  | AI Agent – Email Composer             | ## Loop Over Items of New Purchases 🔁 Batch sender Iterates through each new purchase record for email generation & delivery. |
| AI Agent – Email Composer         | LangChain AI Agent                | Generates personalized HTML thank-you emails using GPT-4o-mini | Loop Over Items of New Purchases         | Send a message (Gmail)                | ## AI Agent – Email Composer 🧩 Personalized HTML email Uses Azure OpenAI (GPT-4o-mini) to craft thank-you emails with access link, password, onboarding tip, and sign-off. |
| Azure OpenAI Chat Model           | LangChain LM Chat Azure OpenAI    | GPT-4o-mini model powering Email Composer     | AI Agent – Email Composer (ai_languageModel) | Structured Output Parser             |                                                                                                        |
| Structured Output Parser          | LangChain Structured Output Parser | Parses email JSON output with Subject and Body | Azure OpenAI Chat Model                  | AI Agent – Email Composer             |                                                                                                        |
| Send a message (Gmail)            | Gmail                            | Sends generated email to customer              | AI Agent – Email Composer                | Append or update row in sheet  For  Tracking | ## Send a message (Gmail) 📨 Email dispatch Sends the AI-generated subject and HTML body to the customer’s email. |
| Append or update row in sheet  For  Tracking | Google Sheets                    | Logs processed purchases and email outcomes    | Send a message (Gmail)                   | Loop Over Items of New Purchases (empty) | ## Append or update row in sheet  For  Tracking 📊 Audit & reporting Logs each processed purchase and email outcome into “Automation Purchase Sheet”. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 7:00 AM IST.

2. **Add Stripe Data Collection Node**  
   - Type: Stripe API (charge resource, getAll)  
   - Credentials: Stripe account (e.g., Techdome)  
   - Configuration: Return all charges. Connect Schedule Trigger output to this node.

3. **Add Filter Node: Filter – Successful Charges**  
   - Condition: `$json.status` equals "succeeded"  
   - Connect Stripe Data Collection output to this node.

4. **Add HTTP Request Node: Get Payment Intent (From Charge)**  
   - Method: GET  
   - URL: `https://api.stripe.com/v1/payment_intents/{{ $json["payment_intent"] }}`  
   - Credentials: Stripe API  
   - Connect Filter output to this node.

5. **Add If Node: If1**  
   - Condition: Check if `{{ $json["payment_details"]["order_reference"] }}` exists and is not empty  
   - True branch: Connect to Get Product Details node  
   - False branch: Connect to Merge Charge + PaymentIntent + Product node for partial data.

6. **Add HTTP Request Node: Get Product Details (from Payment Intent)**  
   - Method: GET  
   - URL: `https://api.stripe.com/v1/products/{{ $json["payment_details"]["order_reference"] }}`  
   - Credentials: Stripe API  
   - Connect If1 true output to this node.

7. **Add Merge Node: Merge Charge + PaymentIntent + Product**  
   - Number of Inputs: 3  
   - Inputs: Get Payment Intent (From Charge) (index 0), If1 false output (index 1), Get Product Details (from Payment Intent) (index 2)  
   - Connect outputs accordingly.

8. **Add Code Node: Merge Logic Format Data**  
   - JavaScript code: Parses inputs, merges charge, intent, and product data into clean JSON object with keys customer_name, email, product_name, etc.  
   - Connect Merge node output to this code node.

9. **Add If Node: Check Required Fields**  
   - Condition: order_reference, product_name, customer_name, and email all not empty  
   - True branch: Connect to AI Agent → Google Sheets Lookup  
   - False branch: Connect to Combine Stripe + Sheet Data node.

10. **Add LangChain AI Agent Node: AI Agent → Google Sheets Lookup**  
    - Input: product_name from previous node  
    - Configure system prompt to instruct AI to search Google Sheets "n8n Automations – Zip Files" for exact Name match, return JSON row or error message.  
    - Use Azure OpenAI GPT-4o-mini as language model.  
    - Parse output with Structured Output Parser using schema with row_number, Name, Google Drive Link, Password, Date.  
    - Connect Check Required Fields true output to this AI Agent node.

11. **Add Google Sheets Node: Get row(s) in sheet in Google Sheets**  
    - Used internally by AI Agent for querying sheet "n8n Automations – Zip Files"  
    - Credentials: Google Sheets OAuth2  
    - No direct connection needed; configured as AI Agent tool.

12. **Add If Node: Check Match Found**  
    - Condition: Output.Name is non-empty  
    - True branch: Connect to Combine Stripe + Sheet Data  
    - False branch: Connect to Combine Stripe + Sheet Data (to handle missing data gracefully)

13. **Add Merge Node: Combine Stripe + Sheet Data**  
    - Inputs: Check Match Found true output and Check Required Fields false output  
    - Joins automation metadata with Stripe purchase data.

14. **Add Google Sheets Node: Get row(s) in sheet – Purchase Sheet Lookup**  
    - Reads "Automation purchase sheet" for existing purchases to avoid duplicates  
    - Credentials: Google Sheets OAuth2  
    - Connect Combine Stripe + Sheet Data output to this node.

15. **Add Code Node: Customer matching**  
    - JavaScript code to match successful charges to automation rows by product name after trimming and removing "[n8n]" suffix.  
    - Filters only succeeded payments, outputs matched purchase data.

16. **Add Merge Node: SQL Combine**  
    - Mode: combineBySql  
    - SQL Query: Select rows from input1 that do not have matching email or customer name in input2 (purchase sheet data), case-insensitive.  
    - Connect Customer matching and Purchase Sheet Lookup nodes to inputs.

17. **Add If Node: Check Automation Exists**  
    - Condition: automationName exists and is non-empty  
    - True branch: Connect to Loop Over Items of New Purchases  
    - False branch: End or error handling.

18. **Add SplitInBatches Node: Loop Over Items of New Purchases**  
    - Default batch size (can be adjusted)  
    - Connect Check Automation Exists true output here.

19. **Add LangChain AI Agent Node: AI Agent – Email Composer**  
    - Input: Customer and purchase details from Loop Over Items  
    - System prompt instructs GPT-4o-mini to generate personalized, friendly, professional HTML thank-you emails including customer name, automation name, Google Drive link, password, purchase date, onboarding tip, sign-off by Rahul Joshi.  
    - Parse output JSON with Subject and Body (HTML).

20. **Add Azure OpenAI Chat Model Node**  
    - Model: GPT-4o-mini  
    - Credential: Azure OpenAI  
    - Connect AI Agent – Email Composer language model input here.

21. **Add Structured Output Parser Node**  
    - Schema: Subject and Body (HTML)  
    - Connect Azure OpenAI Chat Model output here.

22. **Add Gmail Node: Send a message (Gmail)**  
    - Recipient: customerEmail from loop item  
    - Subject and message body from AI Agent output  
    - Credentials: Gmail OAuth2  
    - Connect AI Agent – Email Composer output here.

23. **Add Google Sheets Node: Append or update row in sheet  For  Tracking**  
    - Document: "Automation purchase sheet"  
    - Operation: appendOrUpdate by matching on "name" column  
    - Columns: name, email, price, status, template purchased  
    - Credentials: Google Sheets OAuth2  
    - Connect Gmail node output here.

24. **Loop Back**  
    - Connect Append or update row output back to Loop Over Items of New Purchases main input to continue batch processing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                 | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow uses Azure OpenAI GPT-4o-mini model for AI tasks including Google Sheets lookup and email content generation.                                                      | Azure OpenAI credentials required; model: gpt-4o-mini                                                            |
| Stripe API credentials must have permission to fetch charges, payment intents, and products.                                                                                  | Stripe account credential setup                                                                                   |
| Gmail SMTP OAuth2 credentials needed for sending emails securely.                                                                                                            | Gmail OAuth2 credential                                                                                           |
| Google Sheets OAuth2 credentials must allow reading/writing to "n8n Automations – Zip Files" and "Automation purchase sheet".                                              | Google Sheets API and OAuth2 setup                                                                                 |
| The workflow triggers daily at 7 AM IST; adjust Schedule Trigger node accordingly for timezone or frequency changes.                                                        |                                                                                                                  |
| Sticky notes provide contextual documentation within the workflow for easier maintenance and understanding.                                                                 | Sticky notes linked to each logical block                                                                          |
| Customer matching logic carefully handles product name suffix “[n8n]” removal and trimming to ensure accurate join.                                                        | Implemented in Customer matching code node                                                                         |
| SQL Combine node uses a SQL query to exclude customers already present in purchase logs based on case-insensitive email or name matching.                                  | SQL query embedded in SQL Combine node                                                                             |
| AI email generation follows a strict HTML format to ensure professional and readable emails without markdown or raw URLs.                                                  | See AI Agent – Email Composer node prompt                                                                          |
| For large volumes, consider adjusting batch size in SplitInBatches node to avoid rate limits or timeout issues.                                                             |                                                                                                                  |
| The workflow logs all processed purchases and outcomes for audit and tracking in Google Sheets, enabling manual review and reporting.                                      | Append or update row in sheet  For  Tracking node                                                                  |

---

This documentation provides a complete, structured, and detailed understanding of the "Automated Template Delivery System with Stripe, GPT-4o & Gmail" workflow. It enables technical users to reproduce, modify, and troubleshoot the automation effectively.