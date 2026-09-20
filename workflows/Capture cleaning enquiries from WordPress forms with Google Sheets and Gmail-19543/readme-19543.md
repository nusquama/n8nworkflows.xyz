Capture cleaning enquiries from WordPress forms with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/capture-cleaning-enquiries-from-wordpress-forms-with-google-sheets-and-gmail-19543


# Capture cleaning enquiries from WordPress forms with Google Sheets and Gmail

### 1. Workflow Overview

The **Capture cleaning enquiries from WordPress forms with Google Sheets and Gmail** workflow automates the ingestion, validation, logging, and routing of commercial cleaning enquiries submitted via website contact forms. It accepts incoming form submissions through a webhook, normalizes fields originating from various WordPress plugins, filters out spam or incomplete submissions, logs valid records to Google Sheets, dynamically assigns leads to area owners based on UK postcode prefixes, and manages outbound customer and internal notification emails with a built-in safety preview/test mode.

The operational logic is organized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Receives the raw HTTP POST request from the website form and loads global configuration parameters, routing rules, and company branding constants.
- **1.2 Normalization & Validation:** Parses varying WordPress form payloads into a uniform data structure, generates a human-readable enquiry reference code, and evaluates submission validity against basic anti-spam and completeness criteria.
- **1.3 Owner Assignment & Content Generation:** Matches postcode prefixes to regional area owners, constructs customized HTML/plain-text customer acknowledgement emails containing standard follow-up questions, and builds internal notification summaries.
- **1.4 Persistence & Safety Routing:** Standardizes the enquiry record into row format, appends it to Google Sheets prior to communication attempts, and checks the execution mode (`preview`, `test`, or `live`).
- **1.5 Communication & Response Delivery:** Dispatches emails via Gmail conditionally based on the operating mode and owner configuration, and returns a JSON response to the originating website webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the pipeline by listening for inbound form payloads via webhook and supplying all centralized operational configuration settings.
- **Nodes Involved:** `When Cleaning Enquiry Arrives`, `Configure Enquiry Pipeline`
- **Node Details:**
  - **`When Cleaning Enquiry Arrives`**
    - **Type & Role:** `n8n-nodes-base.webhook` (Webhook Trigger). Acts as the primary workflow entry point.
    - **Configuration:** Configured for `POST` requests at the path `cleaning-enquiry`, using response mode `responseNode`.
    - **Inputs / Outputs:** Input: None (Trigger) | Output: Connects to `Configure Enquiry Pipeline`.
    - **Edge Cases / Failures:** Network timeouts, incorrect endpoint configurations on the WordPress plugin side, or invalid HTTP methods (GET instead of POST).
  - **`Configure Enquiry Pipeline`**
    - **Type & Role:** `n8n-nodes-base.code` (JavaScript Code Node). Establishes the centralized configuration hub for sheet mappings, regional area prefixes, email copy, and safety switches.
    - **Configuration:** Houses configuration variables including `TRACKER_SHEET_ID`, `TRACKER_TAB`, `AREAS`, `QUESTIONS`, and the `TEST_RUN` / `TEST_EMAIL` safety parameters.
    - **Inputs / Outputs:** Input: `When Cleaning Enquiry Arrives` | Output: Connects to `Parse Form Fields`.
    - **Edge Cases / Failures:** JavaScript syntax errors or improperly formatted JSON array structures within configuration definitions.

#### 2.2 Normalization & Validation
- **Overview:** Flattens disparate WordPress form field structures, extracts core contact details and UK postcodes, flags potential spam, and generates a unique reference number.
- **Nodes Involved:** `Parse Form Fields`, `Check If Real Enquiry`, `Handle Spam or Incomplete`
- **Node Details:**
  - **`Parse Form Fields`**
    - **Type & Role:** `n8n-nodes-base.code` (JavaScript Code Node). Normalizes raw form inputs and checks for presence of valid emails, names, messages, and postcodes.
    - **Configuration:** Utilizes recursive parsing functions to flatten multi-dimensional form payloads and regular expressions (`EMAIL_RE`, `PHONE_RE`, `UK_POSTCODE`) to extract data points.
    - **Inputs / Outputs:** Input: `Configure Enquiry Pipeline` | Output: Connects to `Check If Real Enquiry`.
    - **Edge Cases / Failures:** Unrecognized data types or heavily obfuscated spam payloads lacking standard pattern markers.
  - **`Check If Real Enquiry`**
    - **Type & Role:** `n8n-nodes-base.if` (Conditional Router). Determines whether a submission passes validity filters.
    - **Configuration:** Evaluates `{{ $json._looks_real }}` as a boolean condition.
    - **Inputs / Outputs:** Input: `Parse Form Fields` | Output: True branch connects to `Assign Enquiry Owner`; False branch connects to `Handle Spam or Incomplete`.
  - **`Handle Spam or Incomplete`**
    - **Type & Role:** `n8n-nodes-base.code` (JavaScript Code Node). Formats rejected submissions for logging/audit purposes.
    - **Configuration:** Assigns an `_outcome: 'rejected'` status and collects failure reasons.
    - **Inputs / Outputs:** Input: `Check If Real Enquiry` (False branch) | Output: Connects to `Send Webhook Response to Site`.

#### 2.3 Owner Assignment & Content Generation
- **Overview:** Assigns the enquiry to a regional owner using longest-prefix UK postcode matching and generates formatted customer reply and internal notification texts.
- **Nodes Involved:** `Assign Enquiry Owner`, `Compose First Reply`
- **Node Details:**
  - **`Assign Enquiry Owner`**
    - **Type & Role:** `n8n-nodes-base.code` (JavaScript Code Node). Matches sanitized postcodes against configured regional routing rules.
    - **Configuration:** Iterates over configured `AREAS` to find the longest matching prefix, falling back to an office queue if unmatched.
    - **Inputs / Outputs:** Input: `Check If Real Enquiry` (True branch) | Output: Connects to `Compose First Reply`.
  - **`Compose First Reply`**
    - **Type & Role:** `n8n-nodes-base.code` (JavaScript Code Node). Constructs HTML and plain-text body content for both customer acknowledgement and internal team alerts.
    - **Configuration:** Generates responsive email templates incorporating standard follow-up questions defined in the configuration node.
    - **Inputs / Outputs:** Input: `Assign Enquiry Owner` | Output: Connects to `Prepare Sheet Row`.

#### 2.4 Persistence & Safety Routing
- **Overview:** Maps enquiry data into a standardized row layout, writes the record to Google Sheets prior to dispatch, and verifies whether live outbound emails are permitted.
- **Nodes Involved:** `Prepare Sheet Row`, `Append Enquiry to Sheets`, `Check Sending Condition`, `Preview Mode: No Email Sent`
- **Node Details:**
  - **`Prepare Sheet Row`**
    - **Type & Role:** `n8n-nodes-base.code` (JavaScript Code Node). Structures object properties to align with the Google Sheets column schema.
    - **Configuration:** Maps object keys dynamically based on configuration column definitions.
    - **Inputs / Outputs:** Input: `Compose First Reply` | Output: Connects to `Append Enquiry to Sheets`.
  - **`Append Enquiry to Sheets`**
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Google Sheets Integration). Appends the enquiry record as a new row.
    - **Configuration:** Uses expressions for `documentId` (`{{ $('Configure Enquiry Pipeline').first().json.tracker_sheet_id }}`) and `sheetName` (`{{ $('Configure Enquiry Pipeline').first().json.tracker_tab }}`). Cell format is set to `RAW`.
    - **Credentials Required:** Google Sheets OAuth2 / Service Account.
    - **Inputs / Outputs:** Input: `Prepare Sheet Row` | Output: Connects to `Check Sending Condition`.
    - **Edge Cases / Failures:** API rate limits, invalid spreadsheet IDs, missing column headers, or expired OAuth tokens.
  - **`Check Sending Condition`**
    - **Type & Role:** `n8n-nodes-base.if` (Conditional Router). Verifies if live email sending is enabled.
    - **Configuration:** Evaluates `{{ $('Configure Enquiry Pipeline').first().json.send_enabled }}`.
    - **Inputs / Outputs:** Input: `Append Enquiry to Sheets` | Output: True branch connects to `Send Reply to Customer`; False branch connects to `Preview Mode: No Email Sent`.
  - **`Preview Mode: No Email Sent`**
    - **Type & Role:** `n8n-nodes-base.code` (JavaScript Code Node). Handles preview-mode execution flow.
    - **Configuration:** Sets `_outcome: 'preview'` and tags intended recipients without dispatching mail.
    - **Inputs / Outputs:** Input: `Check Sending Condition` (False branch) | Output: Connects to `Send Webhook Response to Site`.

#### 2.5 Communication & Response Delivery
- **Overview:** Dispatches email communications through Gmail (respecting test/live modes and owner email configurations) and returns a completion status to the website webhook.
- **Nodes Involved:** `Send Reply to Customer`, `Verify Owner Email Exists`, `No Owner Email Defined`, `Notify Owner via Email`, `Send Webhook Response to Site`
- **Node Details:**
  - **`Send Reply to Customer`**
    - **Type & Role:** `n8n-nodes-base.gmail` (Gmail Integration). Sends the initial acknowledgment message to the customer.
    - **Configuration:** Configured for HTML email dispatch with dynamic recipient routing (`test_email` vs. customer email), custom sender names, and reply-to headers. Error handling is set to `continueRegularOutput`.
    - **Credentials Required:** Gmail OAuth2.
    - **Inputs / Outputs:** Input: `Check Sending Condition` (True branch) | Output: Connects to `Verify Owner Email Exists`.
    - **Edge Cases / Failures:** Gmail API quota exhaustion, daily sending limits, or invalid recipient addresses.
  - **`Verify Owner Email Exists`**
    - **Type & Role:** `n8n-nodes-base.if` (Conditional Router). Checks if an owner email address has been configured.
    - **Configuration:** Evaluates `{{ $('Compose First Reply').first().json._owner_has_email }}`.
    - **Inputs / Outputs:** Input: `Send Reply to Customer` | Output: True branch connects to `Notify Owner via Email`; False branch connects to `No Owner Email Defined`.
  - **`No Owner Email Defined`**
    - **Type & Role:** `n8n-nodes-base.noOp` (No Operation / Pass-Through). Acts as a routing bypass when no owner email address is present.
    - **Configuration:** Default pass-through parameters.
    - **Inputs / Outputs:** Input: `Verify Owner Email Exists` (False branch) | Output: Connects to `Send Webhook Response to Site`.
  - **`Notify Owner via Email`**
    - **Type & Role:** `n8n-nodes-base.gmail` (Gmail Integration). Sends internal notification alerts to the assigned area owner.
    - **Configuration:** HTML-formatted dispatch supporting test mode redirection. Error handling set to `continueRegularOutput`.
    - **Credentials Required:** Gmail OAuth2.
    - **Inputs / Outputs:** Input: `Verify Owner Email Exists` (True branch) | Output: Connects to `Send Webhook Response to Site`.
  - **`Send Webhook Response to Site`**
    - **Type & Role:** `n8n-nodes-base.respondToWebhook` (Webhook Response Node). Returns the final JSON payload back to the calling website form.
    - **Configuration:** Responds with JSON containing `{ ok: true, ref: ... }`.
    - **Inputs / Outputs:** Inputs: `Handle Spam or Incomplete`, `No Owner Email Defined`, `Notify Owner via Email`, `Preview Mode: No Email Sent` | Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Cleaning Enquiry Arrives` | `n8n-nodes-base.webhook` | Webhook Trigger | None | `Configure Enquiry Pipeline` | ## 01-capture-and-reply<br><br>### How it works<br><br>This workflow captures commercial cleaning enquiries from a website webhook, normalizes the submitted form fields, and filters out spam or incomplete requests. Valid enquiries are assigned to an owner, given an acknowledgement message, logged to Google Sheets, and optionally sent via Gmail to the customer and the assigned owner. All paths return a response to the website, including preview mode, invalid enquiries, and missing-owner-email cases.<br><br>### Setup steps<br><br>- Configure the webhook URL on the website form so enquiries post to the “Enquiry arrives” trigger.<br>- Update the config code node with service areas, owner routing rules, preview/send mode, sheet details, and email sender settings.<br>- Connect Google Sheets credentials for “[cred] Sheets - log the enquiry” and verify the target spreadsheet columns match the generated row shape.<br>- Connect Gmail credentials for the customer reply and owner notification nodes, then test with preview mode before enabling real sending.<br><br>### Customization<br><br>Adjust the form-field aliases, validation rules, postcode-to-owner routing, acknowledgement copy, internal notification text, and preview/send flag to match the business process. |
| `Configure Enquiry Pipeline` | `n8n-nodes-base.code` | Configuration Hub | `When Cleaning Enquiry Arrives` | `Parse Form Fields` | ## 01-capture-and-reply... *(see above)* |
| `Parse Form Fields` | `n8n-nodes-base.code` | Normalization & Validation | `Configure Enquiry Pipeline` | `Check If Real Enquiry` | ## Receive and normalize<br><br>Accepts the incoming website enquiry, loads pipeline configuration, and maps varying WordPress form field names into a consistent structure. |
| `Check If Real Enquiry` | `n8n-nodes-base.if` | Spam & Completeness Router | `Parse Form Fields` | `Assign Enquiry Owner`, `Handle Spam or Incomplete` | ## Validate enquiry<br><br>Checks whether the submission is a real, complete enquiry and diverts spam or incomplete records to a stop path. |
| `Handle Spam or Incomplete` | `n8n-nodes-base.code` | Spam Processing Handler | `Check If Real Enquiry` | `Send Webhook Response to Site` | ## Validate enquiry... *(see above)* |
| `Assign Enquiry Owner` | `n8n-nodes-base.code` | Regional Postcode Routing | `Check If Real Enquiry` | `Compose First Reply` | ## Assign and draft reply<br><br>Routes valid enquiries to the correct owner based on configured postcode rules and creates the customer acknowledgement and internal heads-up content. |
| `Compose First Reply` | `n8n-nodes-base.code` | Email Content Generation | `Assign Enquiry Owner` | `Prepare Sheet Row` | ## Assign and draft reply... *(see above)* |
| `Prepare Sheet Row` | `n8n-nodes-base.code` | Row Schema Standardization | `Compose First Reply` | `Append Enquiry to Sheets` | ## Log enquiry row<br><br>Builds a standardized row for reporting and appends the enquiry to Google Sheets before any outbound email is attempted. |
| `Append Enquiry to Sheets` | `n8n-nodes-base.googleSheets` | Google Sheets Persistence | `Prepare Sheet Row` | `Check Sending Condition` | ## Log enquiry row... *(see above)* |
| `Check Sending Condition` | `n8n-nodes-base.if` | Operational Mode Gate | `Append Enquiry to Sheets` | `Send Reply to Customer`, `Preview Mode: No Email Sent` | ## Check sending mode<br><br>Determines whether the workflow should send emails or stop after logging in preview mode. |
| `Preview Mode: No Email Sent` | `n8n-nodes-base.code` | Preview Mode Handler | `Check Sending Condition` | `Send Webhook Response to Site` | ## Check sending mode... *(see above)* |
| `Send Reply to Customer` | `n8n-nodes-base.gmail` | Outbound Customer Email | `Check Sending Condition` | `Verify Owner Email Exists` | ## Reply to customer<br><br>Sends the initial Gmail response to the customer, then checks whether the assigned owner has an email address for internal notification. |
| `Verify Owner Email Exists` | `n8n-nodes-base.if` | Owner Email Validation | `Send Reply to Customer` | `Notify Owner via Email`, `No Owner Email Defined` | ## Reply to customer... *(see above)* |
| `No Owner Email Defined` | `n8n-nodes-base.noOp` | Bypass for Unassigned Owners | `Verify Owner Email Exists` | `Send Webhook Response to Site` | ## Notify assigned owner<br><br>Either sends the owner notification by Gmail or follows a no-op stop path when no owner email is configured. |
| `Notify Owner via Email` | `n8n-nodes-base.gmail` | Internal Owner Notification | `Verify Owner Email Exists` | `Send Webhook Response to Site` | ## Notify assigned owner... *(see above)* |
| `Send Webhook Response to Site` | `n8n-nodes-base.respondToWebhook` | Webhook HTTP Response | `Handle Spam or Incomplete`, `No Owner Email Defined`, `Notify Owner via Email`, `Preview Mode: No Email Sent` | None | ## Return website response<br><br>Sends the final webhook response back to the website after any branch completes. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create a Webhook Trigger Node:**
   - **Node Type:** `n8n-nodes-base.webhook`
   - **Name:** `When Cleaning Enquiry Arrives`
   - **Parameters:** Set HTTP Method to `POST`, Path to `cleaning-enquiry`, and Response Mode to `Response Node`.

2. **Create the Configuration Code Node:**
   - **Node Type:** `n8n-nodes-base.code`
   - **Name:** `Configure Enquiry Pipeline`
   - **Connection:** Connect `When Cleaning Enquiry Arrives` to this node.
   - **Parameters:** Paste the configuration JavaScript code (defining `TRACKER_SHEET_ID`, `AREAS`, `QUESTIONS`, and safety switches like `TEST_RUN`).

3. **Create the Form Parser Node:**
   - **Node Type:** `n8n-nodes-base.code`
   - **Name:** `Parse Form Fields`
   - **Connection:** Connect `Configure Enquiry Pipeline` to this node.
   - **Parameters:** Add the normalization script that flattens form payloads, extracts postcodes, validates emails, and generates the enquiry reference (`ref`).

4. **Create the Validation IF Node:**
   - **Node Type:** `n8n-nodes-base.if`
   - **Name:** `Check If Real Enquiry`
   - **Connection:** Connect `Parse Form Fields` to this node.
   - **Parameters:** Add a condition checking `{{ $json._looks_real }}` equals `true` (loose validation).

5. **Create the Spam Handler Node:**
   - **Node Type:** `n8n-nodes-base.code`
   - **Name:** `Handle Spam or Incomplete`
   - **Connection:** Connect the **false** output of `Check If Real Enquiry` to this node.
   - **Parameters:** Add the code returning rejected outcomes and reasons.

6. **Create the Owner Assignment Node:**
   - **Node Type:** `n8n-nodes-base.code`
   - **Name:** `Assign Enquiry Owner`
   - **Connection:** Connect the **true** output of `Check If Real Enquiry` to this node.
   - **Parameters:** Add postcode prefix matching logic to assign regional owners and fallback values.

7. **Create the Reply Composition Node:**
   - **Node Type:** `n8n-nodes-base.code`
   - **Name:** `Compose First Reply`
   - **Connection:** Connect `Assign Enquiry Owner` to this node.
   - **Parameters:** Add the templating code generating HTML/plain-text customer acknowledgements and owner notification messages.

8. **Create the Sheet Row Preparation Node:**
   - **Node Type:** `n8n-nodes-base.code`
   - **Name:** `Prepare Sheet Row`
   - **Connection:** Connect `Compose First Reply` to this node.
   - **Parameters:** Add the mapping code matching column headers from the configuration object.

9. **Create the Google Sheets Append Node:**
   - **Node Type:** `n8n-nodes-base.googleSheets`
   - **Name:** `Append Enquiry to Sheets`
   - **Connection:** Connect `Prepare Sheet Row` to this node.
   - **Parameters:** Set Operation to `Append`, Document ID to `={{ $('Configure Enquiry Pipeline').first().json.tracker_sheet_id }}`, and Sheet Name to `={{ $('Configure Enquiry Pipeline').first().json.tracker_tab }}`. Configure OAuth2 credentials.

10. **Create the Sending Mode IF Node:**
    - **Node Type:** `n8n-nodes-base.if`
    - **Name:** `Check Sending Condition`
    - **Connection:** Connect `Append Enquiry to Sheets` to this node.
    - **Parameters:** Add a condition checking `{{ $('Configure Enquiry Pipeline').first().json.send_enabled }}` equals `true`.

11. **Create the Preview Mode Handler Node:**
    - **Node Type:** `n8n-nodes-base.code`
    - **Name:** `Preview Mode: No Email Sent`
    - **Connection:** Connect the **false** output of `Check Sending Condition` to this node.
    - **Parameters:** Add code returning preview outcomes.

12. **Create the Customer Email Node:**
    - **Node Type:** `n8n-nodes-base.gmail`
    - **Name:** `Send Reply to Customer`
    - **Connection:** Connect the **true** output of `Check Sending Condition` to this node.
    - **Parameters:** Set Send To, Subject, and Message HTML using expressions referencing `Compose First Reply` and configuration test settings. Set Error Handling (`OnError`) to `Continue Regular Output`. Configure Gmail OAuth2 credentials.

13. **Create the Owner Email Verification Node:**
    - **Node Type:** `n8n-nodes-base.if`
    - **Name:** `Verify Owner Email Exists`
    - **Connection:** Connect `Send Reply to Customer` to this node.
    - **Parameters:** Check `{{ $('Compose First Reply').first().json._owner_has_email }}` equals `true`.

14. **Create the No-Op Owner Bypass Node:**
    - **Node Type:** `n8n-nodes-base.noOp`
    - **Name:** `No Owner Email Defined`
    - **Connection:** Connect the **false** output of `Verify Owner Email Exists` to this node.

15. **Create the Owner Email Notification Node:**
    - **Node Type:** `n8n-nodes-base.gmail`
    - **Name:** `Notify Owner via Email`
    - **Connection:** Connect the **true** output of `Verify Owner Email Exists` to this node.
    - **Parameters:** Configure recipient, subject, and HTML body expressions. Set Error Handling to `Continue Regular Output`. Configure Gmail OAuth2 credentials.

16. **Create the Webhook Response Node:**
    - **Node Type:** `n8n-nodes-base.respondToWebhook`
    - **Name:** `Send Webhook Response to Site`
    - **Connections:** Connect the outputs of `Handle Spam or Incomplete`, `Preview Mode: No Email Sent`, `No Owner Email Defined`, and `Notify Owner via Email` to this node.
    - **Parameters:** Set Response Body to `={{ JSON.stringify({ ok: true, ref: $('Parse Form Fields').first().json.ref }) }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Commercial Cleaning Enquiry Pipeline workflow structure (`01-capture-and-reply`) | Core n8n workflow designed for WordPress form integration, automated logging, and postcode-based lead routing. |
| Companion workflow ecosystem reference | Pairs with downstream workflows (such as `02 chase and digest`) utilizing the same shared configuration structure. |