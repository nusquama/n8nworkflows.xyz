Capture and reply to cleaning enquiries with webhooks, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/capture-and-reply-to-cleaning-enquiries-with-webhooks--google-sheets-and-gmail-19663


# Capture and reply to cleaning enquiries with webhooks, Google Sheets and Gmail

### 1. Workflow Overview

This workflow captures incoming commercial cleaning enquiries via a webhook, parses and normalizes the payload across various form structures, validates the submission to prevent spam, routes valid enquiries to owners based on UK postcode prefixes, logs the data to Google Sheets, and manages outbound communications (customer acknowledgement and internal notifications) via Gmail with a built-in testing/preview safety switch. Finally, it responds to the website.

- **1.1 Input Reception:** Listens for HTTP POST requests from website contact forms and initializes global configuration settings.
- **1.2 Data Parsing & Validation:** Normalizes form fields into a standardized schema and filters out spam or incomplete submissions using honeypot detection and basic field verification.
- **1.3 Owner Assignment & Content Generation:** Determines geographic territory and assigns an internal owner based on postcode rules, then compiles both HTML and plain-text email templates for the customer reply and internal alert.
- **1.4 Logging & Safety Gate:** Appends a standardized enquiry record to a Google Sheets tracking spreadsheet before inspecting the global safety switch to execute or bypass live email dispatch.
- **1.5 Dispatch & Webhook Response:** Sends emails via Gmail (handling test/preview modes safely) and returns a JSON confirmation response to the website.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Initializes the pipeline and captures incoming form data sent via an HTTP POST request from a website.
- **Nodes Involved:** `When Cleaning Enquiry Arrives`, `Configure Enquiry Pipeline`
- **Node Details:**
  - **When Cleaning Enquiry Arrives**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Listens for incoming POST submissions on path `/cleaning-enquiry`.
    - *Configuration Choices:* HTTP Method: `POST`, Response Mode: `responseNode`.
    - *Input/Output Connections:* Input: None (Trigger); Output: `Configure Enquiry Pipeline`.
    - *Edge Cases / Failure Types:* Timeout or network drops if the origin website cannot reach the n8n instance.
  - **Configure Enquiry Pipeline**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node). Houses global configuration parameters including spreadsheet IDs, owner routing rules, company contact details, and the global testing/preview switch (`TEST_RUN`).
    - *Configuration Choices:* Custom JavaScript returning a configuration object containing `TRACKER_SHEET_ID`, `AREAS`, `QUESTIONS`, and environment state flags (`mode`, `send_enabled`).
    - *Input/Output Connections:* Input: `When Cleaning Enquiry Arrives`; Output: `Parse Form Fields`.
    - *Edge Cases / Failure Types:* JavaScript syntax errors if variables are improperly updated.

---

#### 2.2 Data Parsing & Validation
- **Overview:** Normalizes disparate form field names (from plugins like WPForms, Contact Form 7, etc.) into a consistent structure, checks for spam indicators, and generates a unique reference code.
- **Nodes Involved:** `Parse Form Fields`, `Check If Real Enquiry`, `Handle Spam or Incomplete`
- **Node Details:**
  - **Parse Form Fields**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node). Flatten and map raw payload keys, extract contact data, validate email/phone formats, locate UK postcodes, detect honeypot triggers, and generate a reference code (`ref`).
    - *Configuration Choices:* Custom parsing logic with regex matchers for emails, phone numbers, and UK postcodes.
    - *Input/Output Connections:* Input: `Configure Enquiry Pipeline`; Output: `Check If Real Enquiry`.
    - *Edge Cases / Failure Types:* Malformed JSON payloads resulting in unhandled empty fields.
  - **Check If Real Enquiry**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router). Branches the workflow based on the `_looks_real` boolean flag.
    - *Configuration Choices:* Evaluates `{{ $json._looks_real }}` equals `true`.
    - *Input/Output Connections:* Input: `Parse Form Fields`; Outputs: `Assign Enquiry Owner` (True branch), `Handle Spam or Incomplete` (False branch).
  - **Handle Spam or Incomplete**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node). Flags rejected submissions with an outcome reason.
    - *Configuration Choices:* Appends `_outcome: 'rejected'` and reason strings to the payload.
    - *Input/Output Connections:* Input: `Check If Real Enquiry` (False branch); Output: `Send Webhook Response to Site`.

---

#### 2.3 Owner Assignment & Content Generation
- **Overview:** Matches customer postcodes against configured geographical area rules to assign an internal owner, then compiles formatted customer acknowledgement and internal notification emails.
- **Nodes Involved:** `Assign Enquiry Owner`, `Compose First Reply`
- **Node Details:**
  - **Assign Enquiry Owner**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node). Iterates through defined postcode prefixes (longest prefix match first) to determine the correct territory owner and fallback routing.
    - *Configuration Choices:* JavaScript mapping prefix rules to output `area`, `owner`, and `owner_email`.
    - *Input/Output Connections:* Input: `Check If Real Enquiry` (True branch); Output: `Compose First Reply`.
  - **Compose First Reply**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node). Generates HTML and plain-text representations for both the customer acknowledgement email (including standard follow-up questions) and the internal owner alert.
    - *Configuration Choices:* HTML template string builder injecting customer names, reference numbers, and company details.
    - *Input/Output Connections:* Input: `Assign Enquiry Owner`; Output: `Prepare Sheet Row`.

---

#### 2.4 Logging & Safety Gate
- **Overview:** Transforms the enquiry data into a standardized row structure and writes it to Google Sheets before evaluating whether emails should be dispatched based on the environment mode.
- **Nodes Involved:** `Prepare Sheet Row`, `Append Enquiry to Sheets`, `Check Sending Condition`, `Preview Mode: No Email Sent`
- **Node Details:**
  - **Prepare Sheet Row**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node). Maps properties to exact Google Sheet column headers.
    - *Configuration Choices:* Constructs an object mapping column titles to enquiry values.
    - *Input/Output Connections:* Input: `Compose First Reply`; Output: `Append Enquiry to Sheets`.
  - **Append Enquiry to Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Node). Appends the formatted row to the configured tracking spreadsheet.
    - *Configuration Choices:* Operation: `Append`, Mapping Mode: Auto-map input data, Sheet Name and Document ID dynamically retrieved from configuration.
    - *Credentials Required:* Google Sheets OAuth2 / Service Account.
    - *Input/Output Connections:* Input: `Prepare Sheet Row`; Output: `Check Sending Condition`.
    - *Edge Cases / Failure Types:* API rate limits, authentication expiry, or mismatch between sheet headers and object keys.
  - **Check Sending Condition**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router). Verifies if live email sending is enabled.
    - *Configuration Choices:* Evaluates `{{ $('Configure Enquiry Pipeline').first().json.send_enabled }}` equals `true`.
    - *Input/Output Connections:* Input: `Append Enquiry to Sheets`; Outputs: `Send Reply to Customer` (True branch), `Preview Mode: No Email Sent` (False branch).
  - **Preview Mode: No Email Sent**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node). Captures payloads when running in preview mode to log simulation state.
    - *Input/Output Connections:* Input: `Check Sending Condition` (False branch); Output: `Send Webhook Response to Site`.

---

#### 2.5 Dispatch & Webhook Response
- **Overview:** Sends emails via Gmail (handling test and live overrides) and returns a final JSON response to the website webhook caller.
- **Nodes Involved:** `Send Reply to Customer`, `Verify Owner Email Exists`, `Notify Owner via Email`, `No Owner Email Defined`, `Send Webhook Response to Site`
- **Node Details:**
  - **Send Reply to Customer**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Gmail Node). Sends the automated HTML acknowledgement email to the prospective customer (or test inbox if test mode is active).
    - *Configuration Choices:* Email Type: `HTML`, Subject and Recipient dynamically evaluated with test-mode override checks. Error handling set to `continueRegularOutput`.
    - *Credentials Required:* Gmail OAuth2.
    - *Input/Output Connections:* Input: `Check Sending Condition` (True branch); Output: `Verify Owner Email Exists`.
  - **Verify Owner Email Exists**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router). Checks if the assigned owner has a valid email address configured for internal notifications.
    - *Configuration Choices:* Evaluates `{{ $('Compose First Reply').first().json._owner_has_email }}` equals `true`.
    - *Input/Output Connections:* Input: `Send Reply to Customer`; Outputs: `Notify Owner via Email` (True branch), `No Owner Email Defined` (False branch).
  - **Notify Owner via Email**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Gmail Node). Sends an internal summary notification to the assigned territory owner.
    - *Configuration Choices:* Email Type: `HTML`, Subject and Recipient dynamically evaluated. Error handling set to `continueRegularOutput`.
    - *Credentials Required:* Gmail OAuth2.
    - *Input/Output Connections:* Input: `Verify Owner Email Exists` (True branch); Output: `Send Webhook Response to Site`.
  - **No Owner Email Defined**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (No-Operation / Dummy Node). Serves as a pass-through when no owner email address is set.
    - *Input/Output Connections:* Input: `Verify Owner Email Exists` (False branch); Output: `Send Webhook Response to Site`.
  - **Send Webhook Response to Site**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (Respond to Webhook Node). Returns the final HTTP response payload containing the status and reference ID back to the website.
    - *Configuration Choices:* Respond with: `JSON`, Response Body: `{{ JSON.stringify({ ok: true, ref: $('Parse Form Fields').first().json.ref }) }}`.
    - *Input/Output Connections:* Inputs: `Handle Spam or Incomplete`, `Preview Mode: No Email Sent`, `Notify Owner via Email`, `No Owner Email Defined`; Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Workflow documentation and setup instructions | None | None | ## 01-capture-and-reply... |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Section comment for receive and normalization block | None | None | ## Receive and normalize... |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Section comment for enquiry validation block | None | None | ## Validate enquiry... |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Section comment for owner assignment block | None | None | ## Assign and draft reply... |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Section comment for Google Sheets logging block | None | None | ## Log enquiry row... |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Section comment for sending mode verification | None | None | ## Check sending mode... |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Section comment for customer email reply block | None | None | ## Reply to customer... |
| `Sticky Note7` | `n8n-nodes-base.stickyNote` | Section comment for owner notification block | None | None | ## Notify assigned owner... |
| `Sticky Note8` | `n8n-nodes-base.stickyNote` | Section comment for webhook response block | None | None | ## Return website response... |
| `When Cleaning Enquiry Arrives` | `n8n-nodes-base.webhook` | Receives incoming POST requests from the website form | None | `Configure Enquiry Pipeline` | ## Receive and normalize... |
| `Configure Enquiry Pipeline` | `n8n-nodes-base.code` | Sets up global configuration constants, routing rules, and toggles | `When Cleaning Enquiry Arrives` | `Parse Form Fields` | ## Receive and normalize... |
| `Parse Form Fields` | `n8n-nodes-base.code` | Normalizes form fields, validates formats, detects spam, and generates reference code | `Configure Enquiry Pipeline` | `Check If Real Enquiry` | ## Receive and normalize... |
| `Check If Real Enquiry` | `n8n-nodes-base.if` | Filters out spam or incomplete submissions | `Parse Form Fields` | `Assign Enquiry Owner`, `Handle Spam or Incomplete` | ## Validate enquiry... |
| `Handle Spam or Incomplete` | `n8n-nodes-base.code` | Marks rejected spam entries with an outcome reason | `Check If Real Enquiry` | `Send Webhook Response to Site` | ## Validate enquiry... |
| `Assign Enquiry Owner` | `n8n-nodes-base.code` | Matches postcodes against area routing rules to assign an owner | `Check If Real Enquiry` | `Compose First Reply` | ## Assign and draft reply... |
| `Compose First Reply` | `n8n-nodes-base.code` | Compiles customer acknowledgement and internal owner notification emails | `Assign Enquiry Owner` | `Prepare Sheet Row` | ## Assign and draft reply... |
| `Prepare Sheet Row` | `n8n-nodes-base.code` | Maps enquiry details to match Google Sheet columns | `Compose First Reply` | `Append Enquiry to Sheets` | ## Log enquiry row... |
| `Append Enquiry to Sheets` | `n8n-nodes-base.googleSheets` | Appends the standardized enquiry row to Google Sheets | `Prepare Sheet Row` | `Check Sending Condition` | ## Log enquiry row... |
| `Check Sending Condition` | `n8n-nodes-base.if` | Verifies whether email sending is enabled or if preview mode is active | `Append Enquiry to Sheets` | `Send Reply to Customer`, `Preview Mode: No Email Sent` | ## Check sending mode... |
| `Preview Mode: No Email Sent` | `n8n-nodes-base.code` | Handles preview mode simulation when live sending is disabled | `Check Sending Condition` | `Send Webhook Response to Site` | ## Check sending mode... |
| `Send Reply to Customer` | `n8n-nodes-base.gmail` | Sends the acknowledgement email to the customer | `Check Sending Condition` | `Verify Owner Email Exists` | ## Reply to customer... |
| `Verify Owner Email Exists` | `n8n-nodes-base.if` | Checks if an email address exists for the assigned owner | `Send Reply to Customer` | `Notify Owner via Email`, `No Owner Email Defined` | ## Reply to customer... |
| `No Owner Email Defined` | `n8n-nodes-base.noOp` | Pass-through node when no owner email is configured | `Verify Owner Email Exists` | `Send Webhook Response to Site` | ## Notify assigned owner... |
| `Notify Owner via Email` | `n8n-nodes-base.gmail` | Sends the internal notification email to the assigned owner | `Verify Owner Email Exists` | `Send Webhook Response to Site` | ## Notify assigned owner... |
| `Send Webhook Response to Site` | `n8n-nodes-base.respondToWebhook` | Returns the final JSON response to the website | `Handle Spam or Incomplete`, `Preview Mode: No Email Sent`, `No Owner Email Defined`, `Notify Owner via Email` | None | ## Return website response... |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to recreate the workflow in n8n manually:

1. **Create Webhook Trigger:**
   - Add a **Webhook** node named `When Cleaning Enquiry Arrives`.
   - Set HTTP Method to `POST`, Path to `cleaning-enquiry`, and Response Mode to `Response Node`.
2. **Add Global Configuration:**
   - Add a **Code** node named `Configure Enquiry Pipeline`. Connect it downstream of the Webhook.
   - Paste the global pipeline configuration script (defining `TRACKER_SHEET_ID`, `AREAS`, `QUESTIONS`, and `TEST_RUN`).
3. **Add Form Parser:**
   - Add a **Code** node named `Parse Form Fields`. Connect it downstream of `Configure Enquiry Pipeline`.
   - Paste the payload-flattening and normalization script to extract contact info, validate regex patterns, and generate a reference ID (`ref`).
4. **Add Spam Filter Conditional:**
   - Add an **If** node named `Check If Real Enquiry`. Connect `Parse Form Fields` to it.
   - Set condition to evaluate `{{ $json._looks_real }}` as boolean `true`.
   - Add a **Code** node named `Handle Spam or Incomplete` connected to the *False* output branch.
5. **Add Owner Assignment:**
   - Add a **Code** node named `Assign Enquiry Owner`. Connect it to the *True* output branch of `Check If Real Enquiry`.
   - Paste the postcode prefix matching script.
6. **Add Email Content Composer:**
   - Add a **Code** node named `Compose First Reply`. Connect it downstream of `Assign Enquiry Owner`.
   - Paste the script that builds customer HTML/text emails and owner alert templates.
7. **Add Google Sheets Logging:**
   - Add a **Code** node named `Prepare Sheet Row` downstream of `Compose First Reply`.
   - Add a **Google Sheets** node named `Append Enquiry to Sheets` downstream of `Prepare Sheet Row`.
   - Configure operation as `Append`, set Document ID and Sheet Name dynamically from the configuration node, and connect valid Google Sheets OAuth2 credentials.
8. **Add Sending Mode Check:**
   - Add an **If** node named `Check Sending Condition` downstream of `Append Enquiry to Sheets`.
   - Set condition to check `{{ $('Configure Enquiry Pipeline').first().json.send_enabled }}` equals `true`.
   - Add a **Code** node named `Preview Mode: No Email Sent` connected to the *False* output branch.
9. **Add Customer Email Node:**
   - Add a **Gmail** node named `Send Reply to Customer` connected to the *True* output branch of `Check Sending Condition`.
   - Configure parameters (Recipient, HTML Message, Subject with test-mode evaluation) and connect Gmail credentials. Set error handling (`OnError`) to `Continue Regular Output`.
10. **Add Owner Notification Routing:**
    - Add an **If** node named `Verify Owner Email Exists` downstream of `Send Reply to Customer`.
    - Set condition to evaluate `{{ $('Compose First Reply').first().json._owner_has_email }}` equals `true`.
    - Add a **No-Operation** node named `No Owner Email Defined` connected to the *False* branch.
    - Add a **Gmail** node named `Notify Owner via Email` connected to the *True* branch. Configure parameters similarly and connect Gmail credentials with error handling set to `Continue Regular Output`.
11. **Add Webhook Response Node:**
    - Add a **Respond to Webhook** node named `Send Webhook Response to Site`.
    - Connect the outputs of `Handle Spam or Incomplete`, `Preview Mode: No Email Sent`, `No Owner Email Defined`, and `Notify Owner via Email` into this node.
    - Configure Response Body to return JSON containing `{ ok: true, ref: ... }`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Commercial cleaning enquiry automation pipeline designed to process WordPress and custom website form submissions. | Workflow metadata (`01-capture-and-reply`) |
| Operates in conjunction with pipeline tracking configurations; shares config blocks with companion workflows. | Setup & maintenance guide |