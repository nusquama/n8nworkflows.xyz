Validate wholesale order emails against a Google Sheets price list with Gmail and GPT-4o

https://n8nworkflows.xyz/workflows/validate-wholesale-order-emails-against-a-google-sheets-price-list-with-gmail-and-gpt-4o-19606


# Validate wholesale order emails against a Google Sheets price list with Gmail and GPT-4o

### 1. Workflow Overview

This workflow automates the intake, verification, and logging of wholesale order emails for a food and beverage distributor. It monitors a Gmail inbox for incoming orders, extracts structured line items using an OpenAI LLM agent, validates the items against a live Google Sheets price list, communicates status updates back to customers and the internal order desk, and schedules a weekly management digest.

The logic groups into five distinct functional blocks:

- **1.1 Order Intake & Normalization:** Triggers on new emails matching specific criteria, normalizes email payloads, and routes excessively short or unparsable emails to an error logging lane.
- **1.2 AI Extraction & Price List Matching:** Leverages an OpenAI agent to parse unstructured email bodies into clean JSON order objects, retrieves reference data from Google Sheets (`PriceList`), and executes business logic checks (SKU validity, case-pack multiples, unit price variances, and product active status).
- **1.3 Conditional Order Processing & Communication:** Evaluates validation results to branch the workflow. Problematic orders trigger customer revision requests, while fully validated orders trigger automatic order confirmations. Both paths log records to Google Sheets (`Orders`), alert staff via Telegram, and mark the original email as read.
- **1.4 Weekly Order Desk Digest:** Operates on a cron schedule every Monday morning, aggregates the previous week's orders from Google Sheets, generates an AI-summarized performance report, and distributes it via email and Telegram.
- **1.5 Global Documentation & Metadata:** Provides inline structural documentation via persistent sticky notes detailing operational workflows, integration requirements, and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Order Intake & Normalization
- **Overview:** Monitors the inbox for incoming wholesale order emails, standardizes message metadata, and filters out malformed or empty messages before processing.
- **Nodes Involved:** `When Wholesale Order Email Arrives`, `Normalize Order Email`, `Has Order Details`, `Log Unparsable Order Email`, `Alert Desk About Unparsable Email`
- **Node Details:**
  - **When Wholesale Order Email Arrives**
    - *Type & Role:* `n8n-nodes-base.gmailTrigger` (Trigger). Polls the inbox for unread messages containing "wholesale order" in the subject line.
    - *Configuration:* Poll interval set to every minute. Search query: `is:unread subject:(wholesale order)`.
    - *Inputs / Outputs:* Inputs: None (Trigger). Outputs: Raw Gmail message object.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
    - *Edge Cases:* API rate limits, revoked token credentials, or search query matching unintended emails.
  - **Normalize Order Email**
    - *Type & Role:* `n8n-nodes-base.set` (Data Transformation). Extracts and flattens core message properties.
    - *Configuration:* Assigns variables: `email_id`, `thread_id`, `from_email`, `customer_name`, `subject`, `received_at`, and `raw_text` (truncated to 8,000 characters).
    - *Inputs / Outputs:* Input: Gmail Trigger. Output: Normalized metadata object.
    - *Edge Cases:* Missing sender names or malformed address structures.
  - **Has Order Details**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control). Validates whether the normalized email body contains sufficient character length.
    - *Configuration:* Condition evaluates if `raw_text` length is greater than `20` characters.
    - *Inputs / Outputs:* Input: `Normalize Order Email`. Outputs: True (Proceeds to AI extraction) / False (Proceeds to error logging).
  - **Log Unparsable Order Email**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Destination). Appends error details to an audit log.
    - *Configuration:* Operation: `append`. Sheet Name: `Errors`. Document ID: `1P6Rx0AjH_jpSG4MAa9h6BMkisksO0l0pj_Uyfa0rHK4`.
    - *Inputs / Outputs:* Input: `Has Order Details` (False branch). Output: Appended row confirmation.
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`).
  - **Alert Desk About Unparsable Email**
    - *Type & Role:* `n8n-nodes-base.telegram` (Notification). Notifies staff of an intake failure.
    - *Configuration:* Message text incorporates sender email and subject. Chat ID: `123456789`.
    - *Inputs / Outputs:* Input: `Log Unparsable Order Email`. Output: Sent Telegram message confirmation.
    - *Credentials:* Telegram Bot (`CF Openai Bot`).

#### 2.2 AI Extraction & Price List Matching
- **Overview:** Converts unstructured email text into structured order line items using an OpenAI model, fetches the active catalog from Google Sheets, and cross-references every line item against inventory rules.
- **Nodes Involved:** `AI Extract Order Lines`, `Order Extraction Model`, `Parse Extracted Order`, `Fetch Price List`, `Match Lines To Price List`
- **Node Details:**
  - **AI Extract Order Lines**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent). Orchestrates prompt execution and structural JSON extraction.
    - *Configuration:* System prompt instructs the model to act as a wholesale order parser, returning strict JSON containing customer details, purchase order numbers, delivery dates, notes, and an array of line items (`sku`, `qty`, `unit_price_quoted`).
    - *Inputs / Outputs:* Input: `Has Order Details` (True branch). Output: Model execution text response containing raw JSON string.
  - **Order Extraction Model**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model). Provides underlying chat completion capabilities.
    - *Configuration:* Model name: `gpt-4o-mini`.
    - *Credentials:* OpenAI API (`jonathan`).
  - **Parse Extracted Order**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Execution). Sanitizes the LLM output, strips non-JSON wrappers, and validates schema compliance.
    - *Configuration:* Custom JS extracts substring between `{` and `}`, parses JSON, normalizes SKUs to uppercase, and builds an internal order model.
    - *Inputs / Outputs:* Input: `AI Extract Order Lines`. Output: Parsed order payload object (`parse_ok`, `lines`, etc.).
  - **Fetch Price List**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Retrieval). Loads active distributor catalog pricing and constraints.
    - *Configuration:* Operation: `get` (implicit multi-row fetch). Sheet Name: `PriceList`. Document ID: `1P6Rx0AjH_jpSG4MAa9h6BMkisksO0l0pj_Uyfa0rHK4`.
    - *Inputs / Outputs:* Input: `Parse Extracted Order`. Output: Array of price list rows (`sku`, `product`, `unit_price`, `case_pack`, `status`).
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`).
  - **Match Lines To Price List**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Execution). Validates order items against fetched price list rules.
    - *Configuration:* Checks for missing SKUs, unlisted SKUs, zero or non-case-pack quantities, price variances exceeding `0.01`, and inactive product statuses. Sets status to `needs_review` if issues exist, else `ready_for_fulfilment`.
    - *Inputs / Outputs:* Input: `Fetch Price List`. Output: Validated order object with array of matched lines, total units, order total, and issue counts.

#### 2.3 Conditional Order Processing & Communication
- **Overview:** Routes orders based on validation health, dispatching either a revision request or confirmation email to the customer, logging the transaction to fulfillment tracking sheets, notifying internal staff via Telegram, and marking the original email read.
- **Nodes Involved:** `If Price Or Pack Issue`, `Build Revision Request`, `Email Customer For Revision`, `Alert Order Desk On Telegram1`, `Build Order Confirmation`, `Email Order Confirmation`, `Alert Order Desk On Telegram`, `Log Order To Fulfilment Queue`, `Mark Order Email As Read`
- **Node Details:**
  - **If Price Or Pack Issue**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control). Branches workflow based on whether validation errors were detected.
    - *Configuration:* Condition checks if `issue_count > 0`.
    - *Inputs / Outputs:* Input: `Match Lines To Price List`. Outputs: True (Revision path) / False (Confirmation path).
  - **Build Revision Request**
    - *Type & Role:* `n8n-nodes-base.set` (Data Transformation). Formats email subjects, HTML bodies, and internal tracking fields for problematic orders.
    - *Configuration:* Sets order ID format (`WO-YYYYMMDD-HHmmss`), drafts error itemization tables, and constructs Telegram alert copy.
    - *Inputs / Outputs:* Input: `If Price Or Pack Issue` (True). Output: Formatted payload object.
  - **Email Customer For Revision**
    - *Type & Role:* `n8n-nodes-base.gmail` (Communication). Emails the customer requesting corrected line items.
    - *Configuration:* Recipient: `={{ $json.customer_email }}`. Subject: Dynamic revision prompt.
    - *Inputs / Outputs:* Input: `Build Revision Request`. Output: Sent email confirmation.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
  - **Alert Order Desk On Telegram1**
    - *Type & Role:* `n8n-nodes-base.telegram` (Notification). Alerts staff that an incoming order requires manual correction.
    - *Configuration:* Chat ID: `123456789`. Text: Issue summary string.
    - *Inputs / Outputs:* Input: `Build Revision Request`. Output: Telegram message delivery receipt.
    - *Credentials:* Telegram Bot (`CF Openai Bot`).
  - **Build Order Confirmation**
    - *Type & Role:* `n8n-nodes-base.set` (Data Transformation). Formats confirmation correspondence and fulfillment data for valid orders.
    - *Configuration:* Generates order ID, structured HTML invoice summary, pricing totals, and staff notification strings.
    - *Inputs / Outputs:* Input: `If Price Or Pack Issue` (False). Output: Confirmation payload object.
  - **Email Order Confirmation**
    - *Type & Role:* `n8n-nodes-base.gmail` (Communication). Sends formal confirmation to the customer.
    - *Configuration:* Recipient: `={{ $json.customer_email }}`. Subject: Order confirmation reference and value.
    - *Inputs / Outputs:* Input: `Build Order Confirmation`. Output: Sent email receipt.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
  - **Alert Order Desk On Telegram**
    - *Type & Role:* `n8n-nodes-base.telegram` (Notification). Broadcasts successful order confirmation to internal channels.
    - *Configuration:* Chat ID: `123456789`. Text: Summary of validated units and revenue.
    - *Inputs / Outputs:* Input: `Build Order Confirmation`. Output: Telegram dispatch receipt.
    - *Credentials:* Telegram Bot (`CF Openai Bot`).
  - **Log Order To Fulfilment Queue**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Destination). Appends processed order records to the persistent spreadsheet log.
    - *Configuration:* Operation: `append`. Sheet Name: `Orders`. Document ID: `1P6Rx0AjH_jpSG4MAa9h6BMkisksO0l0pj_Uyfa0rHK4`. Maps all order attributes (`order_id`, `customer`, `status`, `order_total`, etc.).
    - *Inputs / Outputs:* Input: `Build Order Confirmation` or `Build Revision Request`. Output: Appended row receipt.
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`).
  - **Mark Order Email As Read**
    - *Type & Role:* `n8n-nodes-base.gmail` (Inbox Management). Updates email status in Gmail to prevent duplicate processing.
    - *Configuration:* Operation: `markAsRead`. Message ID: `={{ $('Normalize Order Email').first().json.email_id }}`.
    - *Inputs / Outputs:* Input: `Log Order To Fulfilment Queue`. Output: Updated message status confirmation.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).

#### 2.4 Weekly Order Desk Digest
- **Overview:** Triggers weekly on Mondays to compile order metrics, generate an AI summary report, and email/broadcast management analytics.
- **Nodes Involved:** `Weekly Order Desk Digest`, `Read Orders This Week`, `Summarise Order Week`, `AI Write Digest Summary`, `Digest Model`, `Format Digest Email`, `Email Weekly Digest`, `Alert Digest On Telegram`
- **Node Details:**
  - **Weekly Order Desk Digest**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Fires execution on a cron schedule.
    - *Configuration:* Cron Expression: `0 8 * * 1` (Every Monday at 08:00 UTC).
    - *Inputs / Outputs:* Inputs: None (Trigger). Output: Execution timestamp.
  - **Read Orders This Week**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Retrieval). Loads historical order rows for analysis.
    - *Configuration:* Sheet Name: `Orders`. Document ID: `1P6Rx0AjH_jpSG4MAa9h6BMkisksO0l0pj_Uyfa0rHK4`.
    - *Inputs / Outputs:* Input: `Weekly Order Desk Digest`. Output: Full array of logged order rows.
    - *Credentials:* Google Sheets OAuth2 (`Feedback Analyzer Sheets`).
  - **Summarise Order Week**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Execution). Filters database entries down to the past 7 days and computes operational statistics.
    - *Configuration:* Calculates total orders, confirmation rates, revision rates, aggregate units, and gross order values.
    - *Inputs / Outputs:* Input: `Read Orders This Week`. Output: Aggregated statistical object.
  - **AI Write Digest Summary**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent). Generates human-readable executive digest copy.
    - *Configuration:* System prompt enforces strict JSON output containing `digest_subject`, HTML `digest_body` (under 150 words), and concise `tg_text`.
    - *Inputs / Outputs:* Input: `Summarise Order Week`. Output: LLM generated JSON string.
  - **Digest Model**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model). Supplies inference power to the digest agent.
    - *Configuration:* Model name: `gpt-4o-mini`.
    - *Credentials:* OpenAI API (`jonathan`).
  - **Format Digest Email**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Execution). Parses and sanitizes the LLM digest output, providing fallback structures if needed.
    - *Configuration:* Extracts JSON payload from code blocks or raw outputs and merges properties with week summaries.
    - *Inputs / Outputs:* Input: `AI Write Digest Summary`. Output: Cleaned digest object.
  - **Email Weekly Digest**
    - *Type & Role:* `n8n-nodes-base.gmail` (Communication). Delivers the digest email to management.
    - *Configuration:* Recipient: `user@example.com` (Note: default template references `user@example.com`, needs production override to real recipient). Subject: Dynamic digest subject line.
    - *Inputs / Outputs:* Input: `Format Digest Email`. Output: Sent email confirmation.
    - *Credentials:* Gmail OAuth2 (`Gmail Fresh Sep05`).
  - **Alert Digest On Telegram**
    - *Type & Role:* `n8n-nodes-base.telegram` (Notification). Publishes the executive summary to team chat channels.
    - *Configuration:* Chat ID: `123456789`. Text: `={{ $json.tg_text }}`.
    - *Inputs / Outputs:* Input: `Email Weekly Digest`. Output: Telegram transmission receipt.
    - *Credentials:* Telegram Bot (`CF Openai Bot`).

#### 2.5 Global Documentation & Metadata
- **Overview:** Non-executable sticky note nodes providing architectural context, instructions, and maintenance notes directly inside the canvas.
- **Nodes Involved:** `Sticky Note`, `Order intake note`, `Price check note`, `Reply note`, `Digest note`
- **Node Details:**
  - **Sticky Note** (Canvas-wide reference)
    - *Type & Role:* `n8n-nodes-base.stickyNote` (Documentation). Outlines system overview, setup requirements, and extension alternatives.
    - *Configuration:* Dimensions: Width 480px, Height 832px. Content summarizes overall purpose and setup steps.
  - **Order intake note**
    - *Type & Role:* `n8n-nodes-base.stickyNote` (Documentation). Visually frames nodes `When Wholesale Order Email Arrives` through `Normalize Order Email`.
    - *Configuration:* Color tag 7. Width 758px, Height 558px.
  - **Price check note**
    - *Type & Role:* `n8n-nodes-base.stickyNote` (Documentation). Visually frames nodes `AI Extract Order Lines` through `Match Lines To Price List`.
    - *Configuration:* Color tag 7. Width 1256px, Height 414px.
  - **Reply note**
    - *Type & Role:* `n8n-nodes-base.stickyNote` (Documentation). Visually frames nodes `If Price Or Pack Issue` through `Mark Order Email As Read`.
    - *Configuration:* Color tag 7. Width 1398px, Height 718px.
  - **Digest note**
    - *Type & Role:* `n8n-nodes-base.stickyNote` (Documentation). Visually frames nodes `Weekly Order Desk Digest` through `Alert Digest On Telegram`.
    - *Configuration:* Color tag 7. Width 1748px, Height 462px.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Wholesale Order Email Arrives` | `gmailTrigger` | Polls unread wholesale order emails | None | `Normalize Order Email` | ## 09-15-01 validate wholesale order emails against a price list with AI<br><br>### How it works<br><br>Wholesale orders arrive as free text email. This workflow reads each one, extracts the line items with an AI agent, then checks every line against your price list: unknown SKU, wrong unit price, or a quantity that is not a whole case pack. Clean orders get an automatic confirmation and drop into the fulfilment queue. Problem orders get a revision request listing exactly what to fix, before anything ships. Every order is logged to Google Sheets, the desk gets a Telegram alert, and a Monday morning digest reports the week.<br><br>### Setup steps<br><br>- Connect Gmail and set the trigger search filter so it matches your order emails.<br>- Create a Google Sheet with three tabs: PriceList, Orders and Errors, then point the Google Sheets nodes at it.<br>- Fill PriceList with sku, product, unit_price, case_pack and status.<br>- Set the digest address, connect Telegram, then send one test order email.<br><br>### Customization<br><br>Swap Sheets for Airtable or Postgres, Telegram for Slack, or skip the customer reply and just alert the desk.<br>## 1. Order intake<br><br>New wholesale order emails land here. Anything too short to be an order goes straight to the error lane. |
| `Normalize Order Email` | `set` | Normalizes incoming message payload properties | `When Wholesale Order Email Arrives` | `Has Order Details` | ## 1. Order intake<br><br>New wholesale order emails land here. Anything too short to be an order goes straight to the error lane. |
| `Has Order Details` | `if` | Evaluates if email body meets minimum length requirements | `Normalize Order Email` | `AI Extract Order Lines`, `Log Unparsable Order Email` | ## 1. Order intake<br><br>New wholesale order emails land here. Anything too short to be an order goes straight to the error lane. |
| `AI Extract Order Lines` | `agent` | Parses order line items and metadata via LLM agent | `Has Order Details` | `Parse Extracted Order` | ## 2. AI reads the order and checks the price list<br><br>Line items are extracted, then every line is checked for unknown SKUs, wrong unit prices and quantities that are not a full case pack. |
| `Order Extraction Model` | `lmChatOpenAi` | Provides gpt-4o-mini LLM backend for extraction agent | None | `AI Extract Order Lines` | ## 2. AI reads the order and checks the price list<br><br>Line items are extracted, then every line is checked for unknown SKUs, wrong unit prices and quantities that are not a full case pack. |
| `Parse Extracted Order` | `code` | Cleans and structures LLM JSON output | `AI Extract Order Lines` | `Fetch Price List` | ## 2. AI reads the order and checks the price list<br><br>Line items are extracted, then every line is checked for unknown SKUs, wrong unit prices and quantities that are not a full case pack. |
| `Fetch Price List` | `googleSheets` | Retrieves catalog prices and constraints | `Parse Extracted Order` | `Match Lines To Price List` | ## 2. AI reads the order and checks the price list<br><br>Line items are extracted, then every line is checked for unknown SKUs, wrong unit prices and quantities that are not a full case pack. |
| `Match Lines To Price List` | `code` | Validates extracted order lines against price list constraints | `Fetch Price List` | `If Price Or Pack Issue` | ## 2. AI reads the order and checks the price list<br><br>Line items are extracted, then every line is checked for unknown SKUs, wrong unit prices and quantities that are not a full case pack. |
| `If Price Or Pack Issue` | `if` | Branches execution based on validation error presence | `Match Lines To Price List` | `Build Revision Request`, `Build Order Confirmation` | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |
| `Build Revision Request` | `set` | Formats customer revision email and alerts | `If Price Or Pack Issue` | `Email Customer For Revision`, `Log Order To Fulfilment Queue`, `Alert Order Desk On Telegram1` | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |
| `Build Order Confirmation` | `set` | Formats customer confirmation correspondence | `If Price Or Pack Issue` | `Email Order Confirmation`, `Alert Order Desk On Telegram`, `Log Order To Fulfilment Queue` | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |
| `Email Customer For Revision` | `gmail` | Sends error revision request to customer | `Build Revision Request` | None | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |
| `Email Order Confirmation` | `gmail` | Sends formal order confirmation to customer | `Build Order Confirmation` | None | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |
| `Alert Order Desk On Telegram` | `telegram` | Sends confirmation notification to internal chat desk | `Build Order Confirmation` | None | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |
| `Log Order To Fulfilment Queue` | `googleSheets` | Appends processed order transaction to Google Sheets | `Build Revision Request`, `Build Order Confirmation` | `Mark Order Email As Read` | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |
| `Mark Order Email As Read` | `gmail` | Marks original intake email as read | `Log Order To Fulfilment Queue` | None | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |
| `Log Unparsable Order Email` | `googleSheets` | Logs short/malformed emails to Errors sheet | `Has Order Details` | `Alert Desk About Unparsable Email` | ## 1. Order intake<br><br>New wholesale order emails land here. Anything too short to be an order goes straight to the error lane. |
| `Alert Desk About Unparsable Email` | `telegram` | Alerts staff of unparsable incoming email | `Log Unparsable Order Email` | None | ## 1. Order intake<br><br>New wholesale order emails land here. Anything too short to be an order goes straight to the error lane. |
| `Weekly Order Desk Digest` | `scheduleTrigger` | Triggers weekly report schedule every Monday | None | `Read Orders This Week` | ## 4. Weekly order desk digest<br><br>Every Monday morning the week of orders is summarised into one email and a short Telegram message. |
| `Read Orders This Week` | `googleSheets` | Reads historical order rows from sheet | `Weekly Order Desk Digest` | `Summarise Order Week` | ## 4. Weekly order desk digest<br><br>Every Monday morning the week of orders is summarised into one email and a short Telegram message. |
| `Summarise Order Week` | `code` | Computes weekly order metrics and aggregates | `Read Orders This Week` | `AI Write Digest Summary` | ## 4. Weekly order desk digest<br><br>Every Monday morning the week of orders is summarised into one email and a short Telegram message. |
| `AI Write Digest Summary` | `agent` | Generates AI digest report copy | `Summarise Order Week`, `Digest Model` | `Format Digest Email` | ## 4. Weekly order desk digest<br><br>Every Monday morning the week of orders is summarised into one email and a short Telegram message. |
| `Digest Model` | `lmChatOpenAi` | Provides model backend for digest agent | None | `AI Write Digest Summary` | ## 4. Weekly order desk digest<br><br>Every Monday morning the week of orders is summarised into one email and a short Telegram message. |
| `Format Digest Email` | `code` | Sanitizes and formats digest email and Telegram copy | `AI Write Digest Summary` | `Email Weekly Digest` | ## 4. Weekly order desk digest<br><br>Every Monday morning the week of orders is summarised into one email and a short Telegram message. |
| `Email Weekly Digest` | `gmail` | Sends weekly management digest email | `Format Digest Email` | `Alert Digest On Telegram` | ## 4. Weekly order desk digest<br><br>Every Monday morning the week of orders is summarised into one email and a short Telegram system message. |
| `Alert Digest On Telegram` | `telegram` | Publishes weekly digest summary to Telegram | `Email Weekly Digest` | None | ## 4. Weekly order desk digest<br><br>Every Monday morning the week of orders is summarised into one email and a short Telegram system message. |
| `Alert Order Desk On Telegram1` | `telegram` | Alerts order desk of issue-flagged orders needing fixes | `Build Revision Request` | None | ## 3. Reply, alert and log<br><br>Clean orders get a confirmation. Problem orders get a revision request. Every order is logged and the desk hears about it on Telegram. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Triggers and Intake
1. Create a **Schedule Trigger** node named `Weekly Order Desk Digest`. Set cron expression to `0 8 * * 1`.
2. Create a **Gmail Trigger** node named `When Wholesale Order Email Arrives`. Configure poll frequency to every minute, query to `is:unread subject:(wholesale order)`, and connect your Gmail OAuth2 credentials.
3. Create a **Set** node named `Normalize Order Email`. Connect `When Wholesale Order Email Arrives` to it. Configure assignments to extract `email_id`, `thread_id`, `from_email`, `customer_name`, `subject`, `received_at` ($now.toISO()), and `raw_text` (truncated via `String($json.text || $json.snippet || '').slice(0, 8000)`).
4. Create an **If** node named `Has Order Details`. Connect `Normalize Order Email` to it. Configure condition: `={{ String($json.raw_text || '').length }}` greater than `20`.

#### Step 2: Configure Error Handling Branch
1. Create a **Google Sheets** node named `Log Unparsable Order Email`. Connect the `false` output of `Has Order Details` to it. Set operation to `append`, spreadsheet Document ID, sheet name `Errors`, and map `error_id`, `source`, `reason`, `detail`, and `logged_at`. Connect Google Sheets OAuth2 credentials.
2. Create a **Telegram** node named `Alert Desk About Unparsable Email`. Connect it after `Log Unparsable Order Email`. Set target Chat ID and custom notification text. Connect Telegram Bot credentials.

#### Step 3: Configure AI Extraction and Validation Pipeline
1. Create an **AI Agent** node named `AI Extract Order Lines`. Connect the `true` output of `Has Order Details` to its main input. Set prompt text combining email subject, sender, and raw text. Configure the system message to enforce strict JSON extraction of order fields without markdown wrappers.
2. Create an **OpenAI Chat Model** node named `Order Extraction Model`. Connect its output to the `ai_languageModel` input of `AI Extract Order Lines`. Set model name to `gpt-4o-mini` and configure OpenAI API credentials.
3. Create a **Code** node named `Parse Extracted Order`. Connect `AI Extract Order Lines` to it. Add JavaScript to parse raw JSON boundaries (`{` to `}`) and structure line item properties (`sku`, `qty`, `quoted_price`).
4. Create a **Google Sheets** node named `Fetch Price List`. Connect `Parse Extracted Order` to it. Set document ID and sheet name to `PriceList`.
5. Create a **Code** node named `Match Lines To Price List`. Connect `Fetch Price List` to it. Add validation logic checking SKU membership, active status, case pack quantity multiples, and unit price discrepancies.

#### Step 4: Configure Branching Logic and Fulfillment Actions
1. Create an **If** node named `If Price Or Pack Issue`. Connect `Match Lines To Price List` to it. Set condition: `issue_count > 0`.
2. **Revision Path (True branch):**
   - Create a **Set** node named `Build Revision Request`. Set assignment fields for order ID, customer details, status (`needs_review`), error text, and customer email body.
   - Create a **Gmail** node named `Email Customer For Revision` connected to `Build Revision Request`. Set recipient to customer email and map email body/subject.
   - Create a **Telegram** node named `Alert Order Desk On Telegram1` connected to `Build Revision Request`. Set target chat ID and issue text.
3. **Confirmation Path (False branch):**
   - Create a **Set** node named `Build Order Confirmation`. Set assignment fields for order ID, order totals, matched lines string, status (`ready_for_fulfilment`), and confirmation email body.
   - Create a **Gmail** node named `Email Order Confirmation` connected to `Build Order Confirmation`. Set recipient and message parameters.
   - Create a **Telegram** node named `Alert Order Desk On Telegram` connected to `Build Order Confirmation`. Set target chat ID and confirmation alert text.
4. **Merging & Completion:**
   - Create a **Google Sheets** node named `Log Order To Fulfilment Queue`. Connect both `Build Revision Request` and `Build Order Confirmation` outputs to it. Set operation to `append`, sheet name `Orders`, and map all order attributes.
   - Create a **Gmail** node named `Mark Order Email As Read`. Connect it after `Log Order To Fulfilment Queue`. Set operation to `markAsRead` using message ID expression `={{ $('Normalize Order Email').first().json.email_id }}`.

#### Step 5: Configure Weekly Digest Pipeline
1. Create a **Google Sheets** node named `Read Orders This Week`. Connect `Weekly Order Desk Digest` to it. Set sheet name to `Orders`.
2. Create a **Code** node named `Summarise Order Week`. Connect `Read Orders This Week` to it. Add code filtering rows to a 7-day rolling window and calculating totals, values, and revision rates.
3. Create an **AI Agent** node named `AI Write Digest Summary`. Connect `Summarise Order Week` to it. Configure system message to return strict JSON with `digest_subject`, HTML `digest_body`, and `tg_text`.
4. Create an **OpenAI Chat Model** node named `Digest Model`. Connect it to the `ai_languageModel` input of `AI Write Digest Summary`, selecting `gpt-4o-mini`.
5. Create a **Code** node named `Format Digest Email`. Connect `AI Write Digest Summary` to it to parse JSON output.
6. Create a **Gmail** node named `Email Weekly Digest`. Connect it to `Format Digest Email`. Set recipient email (e.g., management address), subject, and HTML body expressions.
7. Create a **Telegram** node named `Alert Digest On Telegram`. Connect it after `Email Weekly Digest`. Set chat ID and Telegram text expression.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Core Spreadsheet Schema Setup | Requires a Google Sheet containing three specific tabs: `PriceList` (columns: `sku`, `product`, `unit_price`, `case_pack`, `status`), `Orders` (tracking all incoming order records), and `Errors` (logging unparsable intake items). |
| Document ID Reference | All Google Sheets nodes point to spreadsheet document ID: `1P6Rx0AjH_jpSG4MAa9h6BMkisksO0l0pj_Uyfa0rHK4`. |
| Default Digest Recipient | The weekly digest email node is pre-configured with a placeholder recipient address (`user@example.com`) which should be updated to a valid distribution list before activation. |