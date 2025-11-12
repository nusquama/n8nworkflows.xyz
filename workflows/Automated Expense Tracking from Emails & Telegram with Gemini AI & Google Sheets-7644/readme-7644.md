Automated Expense Tracking from Emails & Telegram with Gemini AI & Google Sheets

https://n8nworkflows.xyz/workflows/automated-expense-tracking-from-emails---telegram-with-gemini-ai---google-sheets-7644


# Automated Expense Tracking from Emails & Telegram with Gemini AI & Google Sheets

### 1. Workflow Overview

This workflow automates personal and shared finance management by extracting, classifying, and logging expense and budget data from two main input sources: Gmail bank/UPI transaction emails and Telegram messages. It utilizes Google Gemini AI models for natural language processing to parse unstructured text into structured financial data, which is then stored in Google Sheets for tracking and analysis.

The workflow is logically divided into two main input processing pipelines and related data handling blocks:

- **1.1 Telegram Input Processing:** Receives text input from a Telegram bot, uses Gemini AI to parse financial data, classifies messages as budgets or expenses, and logs them into the appropriate Google Sheets tab.

- **1.2 Gmail Input Processing:** Monitors Gmail for bank and UPI transaction alert emails, extracts and identifies the financial institution or app, uses Gemini AI to parse transaction details, classifies transactions by type, and appends them into Google Sheets.

- **1.3 Data Storage & Confirmation:** Appends parsed and classified data into designated sheets within a Google Sheets document, including separate sheets for expenses and budgets. Sends confirmation replies via Telegram when entries are added from Telegram input.

- **1.4 Conditional Routing & Validation:** Various conditional checks determine the classification of data (budget vs. expense, credit vs. debit) to route data correctly and maintain data integrity.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Processing

- **Overview:**  
  This block handles incoming Telegram messages via a trigger, uses Google Gemini AI to parse the text into structured financial data, classifies the data as budget or expense, and routes it accordingly for storage.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Google Gemini Chat Model  
  - Information extraction from telegram input (Chain LLM)  
  - Raw check if the transaction is 'Budget' or 'Expense' (IF)  
  - Budget information extractor (Chain LLM)  
  - Expense information extractor (Chain LLM)  
  - Check if the transaction is 'Budget' or 'Expense' (IF)  
  - Append transaction data to budget sheet (Google Sheets)  
  - Append transaction data to expense sheet (Google Sheets)  
  - Send a confirmation reply to the user (Telegram)  

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram messages  
    - Configuration: Monitors all incoming messages (`updates: ["message"]`)  
    - Inputs: None (trigger)  
    - Outputs: Message JSON object including user text  
    - Failure Modes: Network issues, Telegram API errors, webhook misconfiguration  
    - Version: 1.2  

  - **Google Gemini Chat Model**  
    - Type: Language model node using Google Gemini (PaLM) API  
    - Purpose: To parse Telegram message text for financial data  
    - Configuration: Default options, requires Google Palm API credential  
    - Input: Raw text from Telegram message  
    - Output: AI-generated text response  
    - Failure Modes: API authentication failure, rate limits, malformed input  

  - **Information extraction from telegram input (Chain LLM)**  
    - Type: Chain LLM node with prompt-based parsing  
    - Purpose: Extract structured JSON for budget or expense from Telegram text  
    - Configuration: Custom prompt instructing strict JSON output with classification rules  
    - Key Expression: `text = {{$json.message.text}}`  
    - Inputs: Output from Google Gemini Chat Model  
    - Outputs: JSON text representing budget or expense data  
    - Failure Modes: Parsing errors, prompt misunderstanding, missing fields  

  - **Raw check if the transaction is 'Budget' or 'Expense' (IF)**  
    - Type: IF node, expression-based filter  
    - Purpose: Check parsed JSON field `type` for "budget" to route accordingly  
    - Configuration: Parses JSON from text output, compares `type` field  
    - Inputs: Output from Information extraction node  
    - Outputs: Two branches: `true` for budget, `false` for expense  
    - Failure Modes: JSON parse errors if output is malformed  

  - **Budget information extractor (Chain LLM)**  
    - Type: Chain LLM node  
    - Purpose: Further processes budget type messages to extract detailed fields  
    - Configuration: Uses Gemini API with prompt for budget schema  
    - Input: Parsed budget text  
    - Outputs: Structured budget JSON object  
    - Failure Modes: API errors, missing data fields  

  - **Expense information extractor (Chain LLM)**  
    - Type: Chain LLM node  
    - Purpose: Further processes expense messages to extract detailed fields  
    - Configuration: Similar to budget extractor but for expenses  
    - Inputs: Expense JSON text  
    - Outputs: Structured expense JSON object  
    - Failure Modes: As above  

  - **Check if the transaction is 'Budget' or 'Expense' (IF)**  
    - Type: IF node  
    - Purpose: Confirm the type field in structured data for final routing  
    - Inputs: Outputs from budget or expense extractor  
    - Outputs: Routes to append budget or expense sheets accordingly  
    - Failure Modes: Missing or incorrect type field  

  - **Append transaction data to budget sheet (Google Sheets)**  
    - Type: Google Sheets node  
    - Purpose: Append budget records to the "Budgets" sheet of a Google Sheet document  
    - Configuration: Maps JSON fields to columns: Month, Notes, Category, Updated At, Budget Amount  
    - Inputs: Budget JSON data  
    - Outputs: Confirmation on append success  
    - Failure Modes: Authentication failures, sheet access errors, invalid data types  

  - **Append transaction data to expense sheet (Google Sheets)**  
    - Type: Google Sheets node  
    - Purpose: Append expense records to the "Expenses" sheet  
    - Configuration: Maps all relevant expense fields (Date, Account, Amount, etc.)  
    - Inputs: Expense JSON data  
    - Outputs: Confirmation on append success  
    - Failure Modes: Same as budget append node  

  - **Send a confirmation reply to the user (Telegram)**  
    - Type: Telegram message node  
    - Purpose: Sends a confirmation reply with a link to the finance Google Sheet  
    - Configuration: Uses Telegram API credential, dynamic chatId from trigger message  
    - Inputs: Trigger message context to extract chat ID  
    - Outputs: Telegram message sent  
    - Failure Modes: Telegram API errors, invalid chat ID  

---

#### 2.2 Gmail Input Processing

- **Overview:**  
  This block monitors Gmail for incoming bank or UPI transaction alert emails, identifies the sender and account type, uses Gemini AI to parse transaction details, validates and classifies transactions, and appends them into Google Sheets.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Extract the email only from specified bank/UPI apps or the transactions made from them (Code)  
  - Google Gemini Chat Model3  
  - Generate the structured data from the raw emails (Chain LLM)  
  - Google Gemini Chat Model4  
  - Extract the information and parse it (Information Extractor)  
  - Check if the transaction is 'Credit' or 'Debit' (IF)  
  - Append transaction data to expense sheet1 (Google Sheets)  
  - Append transaction data to expense sheet2 (Google Sheets)  

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail incoming emails  
    - Configuration: Polls every hour by default; no specific filters set but recommended filters for bank alert senders should be configured  
    - Inputs: None (trigger)  
    - Outputs: Raw email metadata and snippet  
    - Failure Modes: Authentication errors, Gmail API limits, polling delays  

  - **Extract the email only from specified bank/UPI apps or the transactions made from them (Code)**  
    - Type: Code node (JavaScript)  
    - Purpose: Parses email sender and body snippet to identify bank or UPI app (HDFC, Indian Overseas Bank, Indian Bank, Google Pay, PhonePe, Paytm)  
    - Configuration: Regex checks on sender address and email body for keywords  
    - Key Expressions: Regex matching on sender email and email body  
    - Inputs: Gmail Trigger email data  
    - Outputs: Object with fields: account, from (sender email), snippet (email body), messageId  
    - Failure Modes: Missing fields in email, unrecognized senders marked as "Other" and skipped  
    - Edge Case: Emails not matching expected senders are ignored (empty output)  

  - **Google Gemini Chat Model3**  
    - Type: Google Gemini LLM node  
    - Purpose: Parse raw email snippet text for structured transaction data  
    - Configuration: Uses Google Palm API credentials, default options  
    - Inputs: Email snippet text  
    - Outputs: AI-generated JSON text with transaction details  
    - Failure Modes: API errors, malformed input  

  - **Generate the structured data from the raw emails (Chain LLM)**  
    - Type: Chain LLM node  
    - Purpose: Parse Gemini AI output to strictly formatted JSON with transaction fields  
    - Configuration: Prompt enforces JSON format with schema fields: date, account, from, to, type, category, description, amount, currency, source, messageId, status  
    - Inputs: Text from Gemini Chat Model3  
    - Outputs: Structured JSON object  
    - Failure Modes: Parsing errors, missing fields fallback to defaults  

  - **Google Gemini Chat Model4**  
    - Type: Google Gemini LLM node  
    - Purpose: Further parse structured JSON text for validation and extraction  
    - Inputs: Output from structured data generation  
    - Outputs: Possibly verified or enhanced structured data  

  - **Extract the information and parse it (Information Extractor)**  
    - Type: Langchain Information Extractor  
    - Purpose: Extract and parse JSON fields with defined schema, ensuring data types and formats  
    - Inputs: JSON text  
    - Outputs: Parsed object with typed fields for transaction  

  - **Check if the transaction is 'Credit' or 'Debit' (IF)**  
    - Type: IF node  
    - Purpose: Branch processing based on transaction type "Credit" or "Debit"  
    - Inputs: Parsed transaction data  
    - Outputs: Two branches for credit and debit  

  - **Append transaction data to expense sheet1 (Google Sheets)**  
    - Type: Google Sheets node  
    - Purpose: Append credit transactions to the "Expenses" sheet  
    - Configuration: Maps all transaction fields with respective column names  
    - Inputs: Credit transaction JSON data  
    - Outputs: Confirmation of append success  

  - **Append transaction data to expense sheet2 (Google Sheets)**  
    - Type: Google Sheets node  
    - Purpose: Append debit transactions to the "Expenses" sheet (separate node, possibly for parallel paths)  
    - Inputs: Debit transaction JSON data  
    - Outputs: Confirmation of append success  

---

#### 2.3 Data Append and Confirmation Flow

- **Overview:**  
  This block appends parsed and validated budget or expense records into the respective Google Sheets tabs and sends Telegram confirmation messages for Telegram-originated inputs.

- **Nodes Involved:**  
  - Append transaction data to budget sheet (Google Sheets)  
  - Append transaction data to expense sheet (Google Sheets)  
  - Append transaction data to expense sheet1 (Google Sheets)  
  - Append transaction data to expense sheet2 (Google Sheets)  
  - Send a confirmation reply to the user (Telegram)  

- **Node Details:**  
  - Appending nodes are configured with explicit column mappings to ensure data integrity.  
  - Google Sheets credentials use OAuth2 with access to the target finance spreadsheet.  
  - Confirmation messages include a static link to the Google Sheet for user convenience.  
  - Potential failures include API authorization issues, sheet permission errors, or schema mismatch.

---

### 3. Summary Table

| Node Name                                       | Node Type                                   | Functional Role                                  | Input Node(s)                                                | Output Node(s)                                                | Sticky Note                                                                                                                         |
|------------------------------------------------|---------------------------------------------|-------------------------------------------------|--------------------------------------------------------------|--------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger                               | Telegram Trigger                            | Receives Telegram messages                       | None                                                         | Google Gemini Chat Model                                      | Takes input from a telegram bot which is connected to the n8n workflow telegram trigger. Gemini AI Parser extracts structured details. |
| Google Gemini Chat Model                       | Langchain LM Chat Google Gemini             | Parses Telegram message text                      | Telegram Trigger                                             | Information extraction from telegram input                    | Same as above                                                                                                                       |
| Information extraction from telegram input    | Chain LLM                                   | Extracts structured JSON for budget/expense     | Google Gemini Chat Model                                     | Raw check if the transaction is 'Budget' or 'Expense'         | Same as above                                                                                                                       |
| Raw check if the transaction is 'Budget' or 'Expense' | IF Node                                   | Checks if parsed type = budget                    | Information extraction from telegram input                  | Budget information extractor, Expense information extractor    | Same as above                                                                                                                       |
| Budget information extractor                   | Chain LLM                                   | Parses detailed budget info                       | Raw check if the transaction is 'Budget' or 'Expense' (true) | Check if the transaction is 'Budget' or 'Expense'             | Same as above                                                                                                                       |
| Expense information extractor                  | Chain LLM                                   | Parses detailed expense info                      | Raw check if the transaction is 'Budget' or 'Expense' (false)| Check if the transaction is 'Budget' or 'Expense'             | Same as above                                                                                                                       |
| Check if the transaction is 'Budget' or 'Expense' | IF Node                                   | Final classification check before appending      | Budget information extractor, Expense information extractor  | Append transaction data to budget sheet, Append transaction data to expense sheet | Same as above                                                                                                                       |
| Append transaction data to budget sheet       | Google Sheets                              | Appends budget records into 'Budgets' sheet      | Check if the transaction is 'Budget' or 'Expense' (true)     | Send a confirmation reply to the user                         | Same as above                                                                                                                       |
| Append transaction data to expense sheet      | Google Sheets                              | Appends expense records into 'Expenses' sheet    | Check if the transaction is 'Budget' or 'Expense' (false)    | Send a confirmation reply to the user                         | Same as above                                                                                                                       |
| Send a confirmation reply to the user         | Telegram                                   | Sends confirmation message to Telegram user      | Append transaction data to budget sheet, Append transaction data to expense sheet | None                                                          | Same as above                                                                                                                       |
| Gmail Trigger                                  | Gmail Trigger                              | Monitors Gmail for incoming emails                | None                                                         | Extract the email only from specified bank/UPI apps           | Gmail Trigger captures new bank/UPI emails. Gemini AI Parser extracts structured details (date, amount, category, etc.)              |
| Extract the email only from specified bank/UPI apps or the transactions made from them | Code Node                          | Identifies account and extracts email snippet    | Gmail Trigger                                                | Google Gemini Chat Model3                                      | Same as above                                                                                                                       |
| Google Gemini Chat Model3                       | Langchain LM Chat Google Gemini             | Parses raw email snippet for transaction data    | Extract the email only from specified bank/UPI apps          | Generate the structured data from the raw emails              | Same as above                                                                                                                       |
| Generate the structured data from the raw emails | Chain LLM                                | Parses Gemini output into structured transaction | Google Gemini Chat Model3                                    | Google Gemini Chat Model4                                     | Same as above                                                                                                                       |
| Google Gemini Chat Model4                       | Langchain LM Chat Google Gemini             | Further processes structured data                 | Generate the structured data from the raw emails             | Extract the information and parse it                          | Same as above                                                                                                                       |
| Extract the information and parse it           | Information Extractor                      | Extracts typed fields from JSON                   | Google Gemini Chat Model4                                    | Check if the transaction is 'Credit' or 'Debit'              | Same as above                                                                                                                       |
| Check if the transaction is 'Credit' or 'Debit' | IF Node                                   | Branches based on transaction type                | Extract the information and parse it                         | Append transaction data to expense sheet1, Append transaction data to expense sheet2 | Same as above                                                                                                                       |
| Append transaction data to expense sheet1      | Google Sheets                              | Appends credit transactions into 'Expenses' sheet | Check if the transaction is 'Credit' or 'Debit' (true)       | None                                                          | Same as above                                                                                                                       |
| Append transaction data to expense sheet2      | Google Sheets                              | Appends debit transactions into 'Expenses' sheet | Check if the transaction is 'Credit' or 'Debit' (false)      | None                                                          | Same as above                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen to all incoming messages (`updates: ["message"]`)  
   - Connect Telegram API credentials  

2. **Create Google Gemini Chat Model Node**  
   - Type: Langchain LM Chat Google Gemini  
   - Connect to Telegram Trigger  
   - Use Google Palm API credentials  
   - Default options  

3. **Create "Information extraction from telegram input" Chain LLM Node**  
   - Type: Chain LLM  
   - Input text expression: `={{ $json.message.text }}`  
   - Prompt: Strict financial parser that outputs exactly one JSON object classifying input as budget or expense with defined schema (see details in overview)  
   - Connect to Gemini Chat Model node  

4. **Create IF Node "Raw check if the transaction is 'Budget' or 'Expense'"**  
   - Condition: Parse JSON from previous node output text, check if `.type` equals `"budget"`  
   - Connect from Information extraction node  
   - True branch to Budget information extractor  
   - False branch to Expense information extractor  

5. **Create Budget information extractor Chain LLM Node**  
   - Type: Chain LLM  
   - Input: text from previous IF true branch  
   - Prompt: Refine and define budget JSON according to schema (Month-Year, Category, Budget Amount, Notes, UpdatedAt)  
   - Connect to next IF node  

6. **Create Expense information extractor Chain LLM Node**  
   - Type: Chain LLM  
   - Input: text from IF false branch  
   - Prompt: Refine and define expense JSON according to schema (Timestamp, Date, Account, From, To, Type, Category, Description, Amount, Currency, Source, MessageId, Status)  
   - Connect to next IF node  

7. **Create IF Node "Check if the transaction is 'Budget' or 'Expense'"**  
   - Condition: Check `.output.type` equals `"budget"`  
   - True branch goes to Append transaction data to budget sheet  
   - False branch goes to Append transaction data to expense sheet  

8. **Create Google Sheets Node "Append transaction data to budget sheet"**  
   - Operation: Append  
   - Document: Your finance Google Sheet with "Budgets" tab  
   - Columns mapping: Month, Notes, Category, Updated At, Budget Amount from JSON output fields  
   - Connect from IF true branch  

9. **Create Google Sheets Node "Append transaction data to expense sheet"**  
   - Operation: Append  
   - Document: Same Google Sheet with "Expenses" tab  
   - Columns mapping: Date, Account, From, To, Type, Category, Description, Amount, Currency, Source, MessageId, Status, Timestamp  
   - Connect from IF false branch  

10. **Create Telegram Node "Send a confirmation reply to the user"**  
    - Text: Confirmation message with static Google Sheet link  
    - Chat ID: Use expression to get from Telegram Trigger message user ID `={{ $('Telegram Trigger').item.json.message.from.id }}`  
    - Connect from both append nodes  

11. **Create Gmail Trigger Node**  
    - Polling: Every hour (or as needed)  
    - Set filters for bank/UPI alert senders (e.g., from: alerts@hdfcbank.net OR ealerts@iobnet.co.in OR alerts@indianbank.in)  

12. **Create Code Node "Extract the email only from specified bank/UPI apps or the transactions made from them"**  
    - JavaScript code to parse sender email and snippet, classify account based on regexes for banks and UPI apps  
    - Filter out unrecognized emails ("Other") by returning empty output  

13. **Create Google Gemini Chat Model3 Node**  
    - Connect from Code node  
    - Use Google Palm API credentials  
    - Default options  

14. **Create Chain LLM Node "Generate the structured data from the raw emails"**  
    - Input: Email snippet text  
    - Prompt: Parse credit/debit alerts into structured JSON transaction object with strict schema  
    - Connect from Gemini Chat Model3  

15. **Create Google Gemini Chat Model4 Node**  
    - Connect from structured data generation node  
    - Further parse/validate the structured JSON  

16. **Create Information Extractor Node "Extract the information and parse it"**  
    - Input: JSON text from Gemini Chat Model4  
    - Schema: Typed schema matching transaction fields  

17. **Create IF Node "Check if the transaction is 'Credit' or 'Debit'"**  
    - Condition: Check `.output.type` field for "Credit" or "Debit"  

18. **Create Google Sheets Node "Append transaction data to expense sheet1"**  
    - Append credit transactions to "Expenses" sheet  
    - Map all transaction fields  
    - Connect from IF true branch  

19. **Create Google Sheets Node "Append transaction data to expense sheet2"**  
    - Append debit transactions to "Expenses" sheet  
    - Map all transaction fields  
    - Connect from IF false branch  

20. **Connect all nodes as per the flow described in section 1**  

21. **Configure all credentials:**  
    - Telegram API credentials (bot token)  
    - Google Palm API (Google Gemini) credentials  
    - Gmail OAuth2 credentials  
    - Google Sheets OAuth2 credentials with access to finance spreadsheet  

22. **Create Google Sheet with tabs:**  
    - "Expenses" with columns matching transaction fields  
    - "Budgets" with columns for monthly budget entries  
    - "Yearly Summary" for roll-up (optional, not explicitly handled here)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automates expense tracking by parsing bank transaction emails (HDFC, Indian Bank, IOB, UPI apps) and Telegram messages, logging them directly into Google Sheets with budget control and alerts.                                                                                                                                                                                                                                                                                                     | Sticky Note on workflow overview                                                                   |
| Recommended Gmail filter example: `from:(alerts@hdfcbank.net OR ealerts@iobnet.co.in OR alerts@indianbank.in)` to capture relevant bank emails only. Add UPI alert sender emails as needed.                                                                                                                                                                                                                                                                                                                         | Gmail Trigger configuration note                                                                   |
| Google Sheets template should have tabs named `Expenses`, `Budgets`, and `Yearly Summary` with appropriate columns as described in the node configurations.                                                                                                                                                                                                                                                                                                                                                   | Google Sheets setup                                                                                 |
| Telegram confirmation messages include a static Google Sheet link: https://docs.google.com/spreadsheets/d/1fDiKZVLB07hqjNh4Zr6d_9t4B3SSZPIv62-AchIUF14/edit?usp=sharing                                                                                                                                                                                                                                                                                                                                           | Telegram confirmation message content                                                             |
| Budget and Expense JSON schemas are strictly enforced via Langchain Chain LLM prompts and structured output parsers to ensure consistent data logging.                                                                                                                                                                                                                                                                                                                                                         | Prompt and output parser design                                                                     |
| The workflow can be extended with alerts for budget overruns and monthly/yearly summaries via additional nodes and scheduling.                                                                                                                                                                                                                                                                                                                                                                                | Potential extension note                                                                            |
| Google Gemini (PaLM) API usage requires credentials with billing enabled; watch for API quotas and costs.                                                                                                                                                                                                                                                                                                                                                                                                       | Credential and API usage note                                                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.