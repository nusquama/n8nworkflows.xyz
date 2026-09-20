Capture and deduplicate website leads in Google Sheets with Slack alerts

https://n8nworkflows.xyz/workflows/capture-and-deduplicate-website-leads-in-google-sheets-with-slack-alerts-17749


# Capture and deduplicate website leads in Google Sheets with Slack alerts

### 1. Workflow Overview

This workflow captures website lead submissions via a webhook, validates and normalizes the incoming contact details, checks for existing records in Google Sheets using the submitter's email address, and handles duplicates by updating the existing row rather than appending a new one. Brand-new leads trigger a Slack notification, while returning leads quietly update their timestamps and engagement counters.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming HTTP POST requests and standardizes unstructured or variably-named form fields into a strict lead schema, followed by format validation.
- **1.2 Webhook Response:** Immediately returns a success acknowledgment or a validation error back to the form submitter.
- **1.3 Duplicate Check & Classification:** Queries Google Sheets to find existing records matching the submitted email and classifies the lead status as new or returning.
- **1.4 Action Routing & Notifications:** Branches execution based on the lead status, appending new leads to Google Sheets and sending a Slack alert, or updating existing rows for returning leads.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** This block listens for incoming HTTP POST requests from website forms, extracts fields regardless of naming conventions, validates the email format, and constructs a uniform JSON payload.
- **Nodes Involved:** `When New Lead Received`, `Normalize Lead Data`, `Check Lead Validity`
- **Node Details:**
  - **When New Lead Received**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Acts as the primary entry point for HTTP POST submissions.
    - *Configuration:* Configured for method `POST` on path `new-lead` with `responseMode` set to `responseNode`.
    - *Key Expressions:* Uses generated webhook path ID `b1a7c0e2-0f11-4c22-9a10-0000000000aa`.
    - *Input/Output:* Output connects to `Normalize Lead Data`.
    - *Edge Cases / Failures:* Timeouts or network interruptions if the form sender takes too long to transmit; malformed JSON payloads handled downstream.
  - **Normalize Lead Data**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution). Standardizes different form-builder field variants into unified parameters (email, name, company, phone, message, source) and executes regex validation for email structure.
    - *Configuration:* Executes custom JavaScript processing input payload arrays.
    - *Key Expressions:* Uses standard regular expressions to validate email syntax (`/^[^\\s@]+@[^\\s@]+\\.[^\\s@]{2,}$/`) and checks against a list of free-mail providers.
    - *Input/Output:* Input from `When New Lead Received`; output connects to `Check Lead Validity`.
    - *Edge Cases / Failures:* Throws syntax or runtime exceptions if the payload structure is entirely non-standard or missing body wrappers.
  - **Check Lead Validity**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Evaluates whether the normalized lead contains a valid email address.
    - *Configuration:* Checks boolean evaluation of `{{ $json.valid }}`.
    - *Key Expressions:* `={{ $json.valid }}`
    - *Input/Output:* Input from `Normalize Lead Data`; true branch connects to `Acknowledge Valid Lead`, false branch connects to `Reject Invalid Lead`.
    - *Edge Cases / Failures:* Type mismatches if `valid` evaluates to non-boolean types (mitigated by loose validation settings).

#### 2.2 Webhook Response
- **Overview:** Responds to the HTTP client immediately with either a confirmation or rejection payload based on validation results.
- **Nodes Involved:** `Acknowledge Valid Lead`, `Reject Invalid Lead`
- **Node Details:**
  - **Acknowledge Valid Lead**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response Node). Sends an HTTP 200 JSON acknowledgment back to the form submitter.
    - *Configuration:* Responds with JSON payload stringifying status and email.
    - *Key Expressions:* `={{ JSON.stringify({ status: 'received', email: $json.email }) }}`
    - *Input/Output:* Input from `Check Lead Validity` (True); output connects to `Search Lead in Sheets`.
    - *Edge Cases / Failures:* Fails if the HTTP connection drops before the response completes.
  - **Reject Invalid Lead**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response Node). Sends an HTTP 400 response with a failure reason.
    - *Configuration:* Response code configured explicitly to `400`.
    - *Key Expressions:* `={{ JSON.stringify({ status: 'rejected', reason: 'a valid email address is required' }) }}`
    - *Input/Output:* Input from `Check Lead Validity` (False); terminal node (no output connections).
    - *Edge Cases / Failures:* Similar network dependency risks as the acknowledgment node.

#### 2.3 Duplicate Check & Classification
- **Overview:** Queries Google Sheets for existing records by email address and determines if the submission is a new lead or a returning contact.
- **Nodes Involved:** `Search Lead in Sheets`, `Classify Lead as New or Returning`, `Check If Lead is New`
- **Node Details:**
  - **Search Lead in Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Lookup). Queries the target Google Sheets document to find existing rows matching the email address.
    - *Configuration:* Operation set to filter rows, looking up value `={{ $json.email }}` in column `email`. Has `alwaysOutputData` enabled to ensure empty sets return data instead of stopping the workflow.
    - *Key Expressions:* `={{ $json.email }}`
    - *Input/Output:* Input from `Acknowledge Valid Lead`; output connects to `Classify Lead as New or Returning`.
    - *Edge Cases / Failures:* Google Sheets API rate limits or authorization revocation.
  - **Classify Lead as New or Returning**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution). Cross-references sheet lookup results against the incoming lead data to calculate visit counts, timestamps (`firstSeen`, `lastSeen`), and status flags.
    - *Configuration:* Executes custom JavaScript processing input arrays.
    - *Key Expressions:* Accesses parent data using `$('Normalize Lead Data').first().json`.
    - *Input/Output:* Input from `Search Lead in Sheets`; output connects to `Check If Lead is New`.
    - *Edge Cases / Failures:* Fails if parent node reference path changes or is unresolved.
  - **Check If Lead is New**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router). Routes workflow execution depending on whether the contact is classified as a new lead.
    - *Configuration:* Evaluates `={{ $json.isNew }}` as a boolean.
    - *Key Expressions:* `={{ $json.isNew }}`
    - *Input/Output:* Input from `Classify Lead as New or Returning`; true branch connects to `Add New Lead to Sheets`, false branch connects to `Update Lead in Sheets`.
    - *Edge Cases / Failures:* Undefined evaluation properties if upstream data schemas change.

#### 2.4 Action Routing & Notifications
- **Overview:** Appends new lead records to the sheet and triggers Slack alerts, or updates existing rows for returning contacts.
- **Nodes Involved:** `Add New Lead to Sheets`, `Update Lead in Sheets`, `Notify New Lead in Slack`
- **Node Details:**
  - **Add New Lead to Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Append). Inserts a brand-new row into the Google Sheets database containing full lead metadata.
    - *Configuration:* Operation set to `append`, mapping object keys to explicit columns (`name`, `email`, `phone`, `source`, `status`, `company`, `message`, `last_seen`, `first_seen`, `times_seen`).
    - *Key Expressions:* Mapped to properties like `={{ $json.name }}`, `={{ $json.email }}`, etc.
    - *Input/Output:* Input from `Check If Lead is New` (True); output connects to `Notify New Lead in Slack`.
    - *Edge Cases / Failures:* Sheet schema drift or API write limits.
  - **Update Lead in Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Upsert). Updates an existing row in Google Sheets when a returning lead submits a form again.
    - *Configuration:* Operation set to `appendOrUpdate`, matching on column `email` and updating tracking fields (`email`, `last_seen`, `times_seen`).
    - *Key Expressions:* `={{ $json.email }}`, `={{ $json.lastSeen }}`, `={{ $json.timesSeen }}`
    - *Input/Output:* Input from `Check If Lead is New` (False); terminal node (no output connections).
    - *Edge Cases / Failures:* Matching column missing from sheet schema.
  - **Notify New Lead in Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging Integration). Posts a formatted notification card to a designated Slack channel for every newly captured lead.
    - *Configuration:* Resource set to send text messages to a selected channel.
    - *Key Expressions:* Uses node-reference expressions like `{{ $('Classify Lead as New or Returning').item.json.name }}` to construct markdown-formatted messages.
    - *Input/Output:* Input from `Add New Lead to Sheets`; terminal node (no output connections).
    - *Edge Cases / Failures:* Invalid Slack OAuth scopes, missing channel permissions, or rate limiting.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Capture website leads into Google Sheets without duplicates<br><br>### How it works<br><br>This workflow receives website lead submissions through a webhook, normalizes the incoming field names, and validates the required lead data. Valid leads are acknowledged immediately, then checked against Google Sheets to determine whether they already exist. New leads are appended to the sheet and announced in Slack, while returning leads update the existing sheet row to avoid duplicates.<br><br>### Setup steps<br><br>- Configure the webhook URL in the website form or lead capture tool that will send submissions to n8n.<br>- Review the field-normalization code so it maps the sender’s form fields to the expected lead fields such as name, email, phone, and source.<br>- Connect Google Sheets credentials and select the spreadsheet, sheet, lookup column, append target, and update target used for lead storage.<br>- Enable the Google Sheets lookup node’s settings required for duplicate detection, such as matching on email and always outputting data when no match is found.<br>- Connect Slack credentials and choose the channel where new-lead alerts should be posted.<br>- Test with one valid new lead, one duplicate lead, and one invalid submission to confirm each branch behaves correctly.<br><br>### Customization<br><br>Adjust the validation rules, duplicate-matching key, Google Sheets columns, and Slack alert message to match the fields and sales process used by the website. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Receive and validate lead<br><br>Captures the incoming website form submission, normalizes differently named form fields into a consistent lead structure, and checks whether the submission contains the required valid data. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Send webhook response<br><br>Returns an immediate webhook response: valid submissions are acknowledged and continue into the lead-processing path, while invalid submissions are rejected and stop here. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Check for duplicates<br><br>Looks up the submitted lead in Google Sheets, then classifies the result as a new lead or a returning/existing lead before branching the workflow. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Append and alert<br><br>Handles the upper new-lead branch by appending the lead to Google Sheets and posting a Slack notification for the newly captured lead. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Update returning lead<br><br>Handles the lower returning-lead branch by updating the existing Google Sheets row instead of creating a duplicate record. |
| `When New Lead Received` | `n8n-nodes-base.webhook` | Webhook Trigger | None | `Normalize Lead Data` | |
| `Normalize Lead Data` | `n8n-nodes-base.code` | Data Normalization | `When New Lead Received` | `Check Lead Validity` | ## Receive and validate lead<br><br>Captures the incoming website form submission, normalizes differently named form fields into a consistent lead structure, and checks whether the submission contains the required valid data. |
| `Check Lead Validity` | `n8n-nodes-base.if` | Conditional Router | `Normalize Lead Data` | `Acknowledge Valid Lead`, `Reject Invalid Lead` | ## Receive and validate lead<br><br>Captures the incoming website form submission, normalizes differently named form fields into a consistent lead structure, and checks whether the submission contains the required valid data. |
| `Acknowledge Valid Lead` | `n8n-nodes-base.respondToWebhook` | Response Node | `Check Lead Validity` | `Search Lead in Sheets` | ## Send webhook response<br><br>Returns an immediate webhook response: valid submissions are acknowledged and continue into the lead-processing path, while invalid submissions are rejected and stop here. |
| `Reject Invalid Lead` | `n8n-nodes-base.respondToWebhook` | Response Node | `Check Lead Validity` | None | ## Send webhook response<br><br>Returns an immediate webhook response: valid submissions are acknowledged and continue into the lead-processing path, while invalid submissions are rejected and stop here. |
| `Search Lead in Sheets` | `n8n-nodes-base.googleSheets` | Data Lookup | `Acknowledge Valid Lead` | `Classify Lead as New or Returning` | ## Check for duplicates<br><br>Looks up the submitted lead in Google Sheets, then classifies the result as a new lead or a returning/existing lead before branching the workflow. |
| `Classify Lead as New or Returning` | `n8n-nodes-base.code` | Data Classification | `Search Lead in Sheets` | `Check If Lead is New` | ## Check for duplicates<br><br>Looks up the submitted lead in Google Sheets, then classifies the result as a new lead or a returning/existing lead before branching the workflow. |
| `Check If Lead is New` | `n8n-nodes-base.if` | Conditional Router | `Classify Lead as New or Returning` | `Add New Lead to Sheets`, `Update Lead in Sheets` | ## Check for duplicates<br><br>Looks up the submitted lead in Google Sheets, then classifies the result as a new lead or a returning/existing lead before branching the workflow. |
| `Add New Lead to Sheets` | `n8n-nodes-base.googleSheets` | Data Append | `Check If Lead is New` | `Notify New Lead in Slack` | ## Append and alert<br><br>Handles the upper new-lead branch by appending the lead to Google Sheets and posting a Slack notification for the newly captured lead. |
| `Update Lead in Sheets` | `n8n-nodes-base.googleSheets` | Data Upsert | `Check If Lead is New` | None | ## Update returning lead<br><br>Handles the lower returning-lead branch by updating the existing Google Sheets row instead of creating a duplicate record. |
| `Notify New Lead in Slack` | `n8n-nodes-base.slack` | Messaging Integration | `Add New Lead to Sheets` | None | ## Append and alert<br><br>Handles the upper new-lead branch by appending the lead to Google Sheets and posting a Slack notification for the newly captured lead. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually inside n8n:

1. **Create Webhook Trigger (`When New Lead Received`)**
   - Add a **Webhook** node.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `new-lead`.
   - Set **Response Mode** to `Response Node`.

2. **Add Normalization Logic (`Normalize Lead Data`)**
   - Add a **Code** node connected downstream of the Webhook.
   - Insert JavaScript code that maps raw payload keys (`email`, `name`, `company`, `phone`, `message`, `source`) into standardized fields, validates email regex, and outputs a single normalized JSON array containing properties like `valid`, `email`, `name`, `company`, `phone`, `message`, `source`, `domain`, `isBusinessEmail`, and `receivedAt`.

3. **Add Validation Router (`Check Lead Validity`)**
   - Add an **If** node connected to the Code node output.
   - Set condition: evaluate `={{ $json.valid }}` as boolean `true`.

4. **Configure Response Handlers (`Acknowledge Valid Lead` & `Reject Invalid Lead`)**
   - Connect the **True** branch of the If node to a **Respond to Webhook** node (`Acknowledge Valid Lead`). Set response body to: `={{ JSON.stringify({ status: 'received', email: $json.email }) }}`.
   - Connect the **False** branch of the If node to another **Respond to Webhook** node (`Reject Invalid Lead`). Set response code to `400` and response body to: `={{ JSON.stringify({ status: 'rejected', reason: 'a valid email address is required' }) }}`.

5. **Perform Google Sheets Lookup (`Search Lead in Sheets`)**
   - Connect the output of `Acknowledge Valid Lead` to a **Google Sheets** node.
   - Configure **Credentials**: Provide valid Google Sheets OAuth2 credentials.
   - Configure **Operation**: Set to `Lookup` (or filter rows).
   - Select your target Document and Worksheet (`Leads`).
   - Add filter criteria: Lookup column `email` matched against `={{ $json.email }}`.
   - Enable **Always Output Data** in node options.

6. **Classify Lead Status (`Classify Lead as New or Returning`)**
   - Add a **Code** node connected after the Google Sheets lookup.
   - Insert JavaScript code comparing the lookup results against the incoming lead data (`Normalize Lead Data` reference).
   - Compute `timesSeen`, `firstSeen`, `lastSeen`, and boolean `isNew`.

7. **Add Status Routing Router (`Check If Lead is New`)**
   - Add an **If** node connected after the classification code node.
   - Set condition: evaluate `={{ $json.isNew }}` as boolean `true`.

8. **Configure Lead Append (`Add New Lead to Sheets`)**
   - Connect the **True** branch of the classification If node to a **Google Sheets** node.
   - Set **Operation** to `Append`.
   - Select target Document and Worksheet (`Leads`).
   - Map columns explicitly:
     - `email` $\rightarrow$ `={{ $json.email }}`
     - `name` $\rightarrow$ `={{ $json.name }}`
     - `company` $\rightarrow$ `={{ $json.company }}`
     - `phone` $\rightarrow$ `={{ $json.phone }}`
     - `source` $\rightarrow$ `={{ $json.source }}`
     - `message` $\rightarrow$ `={{ $json.message }}`
     - `first_seen` $\rightarrow$ `={{ $json.firstSeen }}`
     - `last_seen` $\rightarrow$ `={{ $json.lastSeen }}`
     - `times_seen` $\rightarrow$ `={{ $json.timesSeen }}`
     - `status` $\rightarrow$ `={{ $json.status }}`

9. **Configure Lead Update (`Update Lead in Sheets`)**
   - Connect the **False** branch of the classification If node to a **Google Sheets** node.
   - Set **Operation** to `Append or Update`.
   - Select target Document and Worksheet (`Leads`).
   - Configure **Matching Columns** to `email`.
   - Map update columns:
     - `email` $\rightarrow$ `={{ $json.email }}`
     - `last_seen` $\rightarrow$ `={{ $json.lastSeen }}`
     - `times_seen` $\rightarrow$ `={{ $json.timesSeen }}`

10. **Configure Slack Notifications (`Notify New Lead in Slack`)**
    - Connect the output of `Add New Lead to Sheets` to a **Slack** node.
    - Configure **Credentials**: Provide valid Slack OAuth2 or Bot Token credentials.
    - Set resource to `Message` and action to `Post`.
    - Select target channel ID.
    - Insert markdown-formatted template text referencing expression data from the classification node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets Setup Requirement | Must contain a worksheet named `Leads` with exact header columns: `email, name, company, phone, source, message, first_seen, last_seen, times_seen, status`. |