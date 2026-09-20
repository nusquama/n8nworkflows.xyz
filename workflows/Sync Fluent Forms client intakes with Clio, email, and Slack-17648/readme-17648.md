Sync Fluent Forms client intakes with Clio, email, and Slack

https://n8nworkflows.xyz/workflows/sync-fluent-forms-client-intakes-with-clio--email--and-slack-17648


# Sync Fluent Forms client intakes with Clio, email, and Slack

### 1. Workflow Overview

This workflow synchronizes incoming client intake submissions from web forms (such as Fluent Forms) into the Clio legal practice management platform. It handles deduplication of contacts, creates associated legal matters, routes internal notifications to staff via email and Slack, and executes a compliance check before sending an optional confirmation message to the client.

The workflow logic is divided into the following functional blocks:
- **1.1 Input Reception & Validation:** Captures incoming web form payloads via webhook and validates/normalizes required intake fields.
- **1.2 Clio Contact Lookup & Decision:** Searches Clio for existing contacts by email to prevent duplicates and branches execution based on whether a match is found.
- **1.3 Contact Resolution Handling:** Extracts existing contact IDs or creates new Person records in Clio.
- **1.4 Matter Creation & Status Routing:** Creates a new Clio Matter linked to the contact and branches based on API success or failure.
- **1.5 Success Branch - Staff Notifications & Client Confirmation:** Generates staff summaries, dispatches email notifications, posts optional Slack alerts, runs compliance checks, and sends client confirmation emails.
- **1.6 Failure & Error Handling:** Manages Clio API rejections, returns HTTP error responses to form callers, and utilizes an unhandled exception safety net.

---

### 2. Block-by-Block Analysis

---

### 2.1 Input Reception & Validation

#### Overview
This block receives the raw webhook submission, parses the incoming data payload, and validates mandatory fields while normalizing text formatting, practice areas, and phone numbers.

#### Nodes Involved
- `When New Client Intake Form Submitted`
- `Validate And Normalize Intake Data`

#### Node Details

- **When New Client Intake Form Submitted**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (Webhook Trigger). Acts as the public entry point for incoming form submissions.
  - **Configuration:** Configured to accept HTTP `POST` requests on path `client-intake` with `responseMode` set to `responseNode`.
  - **Key Expressions:** None (trigger node).
  - **Connections:** Output connects to `Validate And Normalize Intake Data`.
  - **Version Requirements:** Version 2.1.
  - **Edge Cases & Failure Types:** Unauthenticated or improperly formatted external payloads; handled downstream or via HTTP server limits.

- **Validate And Normalize Intake Data**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution). Validates required schema fields (`full_name`, `email`, `matter_type`) and normalizes telephone numbers to E.164 format and practice areas to standardized labels.
  - **Configuration:** Custom JavaScript parsing `body` or direct JSON inputs. Throws explicit errors if mandatory fields are blank.
  - **Key Expressions:** `$input.first().json.body || $input.first().json`
  - **Connections:** Input from `When New Client Intake Form Submitted`; output connects to `Fetch Existing Clio Contact`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Missing required fields throws a runtime error caught by the global Error Trigger.

---

### 2.2 Clio Contact Lookup & Decision

#### Overview
Performs an API lookup against Clio using the normalized email address to determine if a contact already exists within the system.

#### Nodes Involved
- `Fetch Existing Clio Contact`
- `Determine If Contact Exists`
- `If Contact Exists In Clio`

#### Node Details

- **Fetch Existing Clio Contact**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP Request). Queries the Clio v4 Contacts API endpoint by email.
  - **Configuration:** Generic HTTP Header Authentication using Bearer tokens. Continues execution on regular output even if errors occur (`onError: continueRegularOutput`).
  - **Key Expressions:** `={{ $vars.CLIO_BASE_URL }}/api/v4/contacts.json?query={{ $json.email }}&fields=id,name,primary_email_address`
  - **Connections:** Input from `Validate And Normalize Intake Data`; output connects to `Determine If Contact Exists`.
  - **Credentials:** `Clio API (Bearer token)` (HTTP Header Auth).
  - **Version Requirements:** Version 4.4.
  - **Edge Cases & Failure Types:** Rate limiting or network timeouts; configured to treat errors as non-existent contacts to prevent blocking leads.

- **Determine If Contact Exists**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution). Analyzes the Clio search response array to check for an exact primary email match.
  - **Configuration:** Custom JavaScript comparing list items against normalized intake email.
  - **Key Expressions:** `$('Validate And Normalize Intake Data').first().json`
  - **Connections:** Input from `Fetch Existing Clio Contact`; output connects to `If Contact Exists In Clio`.
  - **Version Requirements:** Version 2.

- **If Contact Exists In Clio**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Branching). Evaluates the boolean result of contact existence.
  - **Configuration:** Checks if `{{ $json.contact_exists }}` is true.
  - **Key Expressions:** `={{ $json.contact_exists }}`
  - **Connections:** Input from `Determine If Contact Exists`. True branch connects to `Extract Existing Contact Id`; False branch connects to `Create New Clio Contact`.
  - **Version Requirements:** Version 2.3.

---

### 2.3 Contact Resolution Handling

#### Overview
Processes the contact path by either passing an existing contact identifier forward or provisioning a brand-new Person contact inside Clio.

#### Nodes Involved
- `Extract Existing Contact Id`
- `Create New Clio Contact`
- `Extract New Contact Id`

#### Node Details

- **Extract Existing Contact Id**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution). Maps the existing contact reference forward as `contact_id`.
  - **Configuration:** Simple JavaScript object transformation.
  - **Connections:** Input from `If Contact Exists In Clio` (True branch); output connects to `Create New Matter In Clio`.
  - **Version Requirements:** Version 2.

- **Create New Clio Contact**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP Request). Sends a `POST` request to the Clio Contacts API to create a new Person record.
  - **Configuration:** JSON payload mapping full name, email addresses, and phone numbers. Continues on error output.
  - **Key Expressions:** `={{ JSON.stringify({ data: { type: 'Person', name: $json.full_name, email_addresses: [{ address: $json.email, name: 'Work' }], phone_numbers: $json.phone ? [{ number: $json.phone, name: 'Mobile' }] : [] } }) }}`
  - **Connections:** Input from `If Contact Exists In Clio` (False branch); output connects to `Extract New Contact Id`.
  - **Credentials:** `Clio API (Bearer token)` (HTTP Header Auth).
  - **Version Requirements:** Version 4.4.

- **Extract New Contact Id**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution). Parses the newly created contact response and extracts the assigned identifier.
  - **Configuration:** Validates the presence of `data.id`, throwing an explicit error if missing.
  - **Connections:** Input from `Create New Clio Contact`; output connects to `Create New Matter In Clio`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Clio returning an empty ID payload throws a descriptive error.

---

### 2.4 Matter Creation & Status Routing

#### Overview
Creates a new legal matter in Clio linked to the resolved contact ID, evaluating whether the API request succeeded before proceeding.

#### Nodes Involved
- `Create New Matter In Clio`
- `If Matter Creation Successful`

#### Node Details

- **Create New Matter In Clio**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP Request). Sends a `POST` request to create a Matter entity in Clio.
  - **Configuration:** JSON payload mapping client ID, practice area name, description, and open status.
  - **Key Expressions:** `={{ JSON.stringify({ data: { client: { id: $json.contact_id }, practice_area: { name: $json.practice_area }, description: $json.case_details || ($json.full_name + ' — ' + $json.practice_area), status: 'Open' } }) }}`
  - **Connections:** Inputs converge from `Extract Existing Contact Id` and `Extract New Contact Id`; output connects to `If Matter Creation Successful`.
  - **Credentials:** `Clio API (Bearer token)` (HTTP Header Auth).
  - **Version Requirements:** Version 4.4.

- **If Matter Creation Successful**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Branching). Verifies if the Clio matter creation returned a valid data object.
  - **Configuration:** Checks if `{{ $json.data }}` is not empty.
  - **Key Expressions:** `={{ $json.data }}`
  - **Connections:** Input from `Create New Matter In Clio`. True branch connects to `Build PM Notification Message` and `Respond Success To Intake Form`; False branch connects to `Build Error Response Payload`.
  - **Version Requirements:** Version 2.3.

---

### 2.5 Success Branch - Staff Notifications & Client Confirmation

#### Overview
Handles post-success operations by aggregating intake metadata, dispatching staff emails and optional Slack alerts, validating bar compliance guardrails, and sending confirmation emails to clients.

#### Nodes Involved
- `Build PM Notification Message`
- `Send Staff Email Notification`
- `If Slack Webhook Configured`
- `Post Slack Alert To Staff`
- `Skip Slack Notification`
- `Build Client Confirmation Message`
- `Execute Bar Compliance Check`
- `If Compliance Approved`
- `Send Confirmation Email`
- `Skip Confirmation Email`
- `Respond Success To Intake Form`

#### Node Details

- **Build PM Notification Message**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution). Assembles a unified operational routing summary payload for staff.
  - **Configuration:** Extracts values across previous node contexts using `.first().json`.
  - **Connections:** Input from `If Matter Creation Successful` (True branch); outputs connect to `Send Staff Email Notification` and `Build Client Confirmation Message`.
  - **Version Requirements:** Version 2.

- **Send Staff Email Notification**
  - **Type & Technical Role:** `n8n-nodes-base.emailSend` (Email Send). Emails staff a plain text operational summary and direct link to the Clio matter record.
  - **Configuration:** Plain text email referencing environment variables for sender/recipient addresses.
  - **Key Expressions:** `toEmail: {{ $vars.FIRM_EMAIL }}`, `fromEmail: {{ $vars.FIRM_FROM_EMAIL }}`, text body containing intake properties.
  - **Connections:** Input from `Build PM Notification Message`; output connects to `If Slack Webhook Configured`.
  - **Credentials:** `Email account (SMTP)` (SMTP).
  - **Version Requirements:** Version 2.

- **If Slack Webhook Configured**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Branching). Checks if the optional Slack webhook URL environment variable has been defined.
  - **Configuration:** Evaluates whether `{{ $vars.FIRM_SLACK_WEBHOOK_URL }}` is not empty.
  - **Key Expressions:** `={{ $vars.FIRM_SLACK_WEBHOOK_URL }}`
  - **Connections:** Input from `Send Staff Email Notification`. True branch connects to `Post Slack Alert To Staff`; False branch connects to `Skip Slack Notification`.
  - **Version Requirements:** Version 2.3.

- **Post Slack Alert To Staff**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP Request). Posts a structured Slack Block Kit message to the configured Incoming Webhook.
  - **Configuration:** Sends a JSON payload containing message blocks with intake details.
  - **Key Expressions:** `={{ $vars.FIRM_SLACK_WEBHOOK_URL }}`
  - **Connections:** Input from `If Slack Webhook Configured` (True branch); has no downstream outputs.
  - **Version Requirements:** Version 4.4.

- **Skip Slack Notification**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` (No Operation). Placeholder node executed when no Slack webhook URL is configured.
  - **Connections:** Input from `If Slack Webhook Configured` (False branch); has no downstream outputs.
  - **Version Requirements:** Version 1.

- **Build Client Confirmation Message**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution). Prepares the initial draft message content for the client acknowledgement email.
  - **Configuration:** Custom JavaScript generating a personalized text greeting based on intake parameters.
  - **Connections:** Input from `Build PM Notification Message`; output connects to `Execute Bar Compliance Check`.
  - **Version Requirements:** Version 2.

- **Execute Bar Compliance Check**
  - **Type & Technical Role:** `n8n-nodes-base.executeWorkflow` (Execute Workflow). Invokes a shared sub-workflow (Bar-Compliance Guardrail) to verify opt-out status and append mandatory legal disclaimers.
  - **Configuration:** Dynamically references target workflow ID from variables and maps input parameters (`channel`, `message`, `matter_id`, `recipient`, `template_id`).
  - **Key Expressions:** Workflow ID: `={{ $vars.GUARDRAIL_WORKFLOW_ID }}`
  - **Connections:** Input from `Build Client Confirmation Message`; output connects to `If Compliance Approved`.
  - **Version Requirements:** Version 1.3.

- **If Compliance Approved**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Branching). Evaluates whether the guardrail sub-workflow approved the outbound message.
  - **Configuration:** Checks if `{{ $json.approved }}` is true.
  - **Key Expressions:** `={{ $json.approved }}`
  - **Connections:** Input from `Execute Bar Compliance Check`. True branch connects to `Send Confirmation Email`; False branch connects to `Skip Confirmation Email`.
  - **Version Requirements:** Version 2.3.

- **Send Confirmation Email**
  - **Type & Technical Role:** `n8n-nodes-base.emailSend` (Email Send). Sends the compliance-approved message (including required disclaimers) to the client.
  - **Configuration:** Plain text email mapping message output and recipient variables.
  - **Key Expressions:** Text: `={{ $json.message_out }}`, To: `={{ $json.recipient }}`
  - **Connections:** Input from `If Compliance Approved` (True branch); has no downstream outputs.
  - **Credentials:** `Email account (SMTP)` (SMTP).
  - **Version Requirements:** Version 2.

- **Skip Confirmation Email**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` (No Operation). Placeholder node used when compliance checks fail or recipients have opted out.
  - **Connections:** Input from `If Compliance Approved` (False branch); has no downstream outputs.
  - **Version Requirements:** Version 1.

- **Respond Success To Intake Form**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook). Returns an HTTP success payload containing contact and matter IDs back to the original form submission endpoint.
  - **Configuration:** Responds with JSON containing status, contact ID, and matter ID.
  - **Key Expressions:** `={{ { status: 'success', contact_id: $('If Matter Creation Successful').first().json.data.client.id, matter_id: $('If Matter Creation Successful').first().json.data.id } }}`
  - **Connections:** Input from `If Matter Creation Successful` (True branch); has no downstream outputs.
  - **Version Requirements:** Version 1.1.

---

### 2.6 Failure & Error Handling

#### Overview
Manages API rejections during matter creation, dispatches staff failure alerts, responds with graceful error codes to web form clients, and catches unhandled workflow exceptions.

#### Nodes Involved
- `Build Error Response Payload`
- `Send Clio Failure Notification`
- `Respond Error To Intake Form`
- `When Workflow Error Occurs`
- `Notify Staff Of Workflow Error`

#### Node Details

- **Build Error Response Payload**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution). Formats error details when Clio rejects matter creation requests.
  - **Configuration:** Wraps error responses into a standardized summary object.
  - **Connections:** Input from `If Matter Creation Successful` (False branch); outputs connect to `Send Clio Failure Notification` and `Respond Error To Intake Form`.
  - **Version Requirements:** Version 2.

- **Send Clio Failure Notification**
  - **Type & Technical Role:** `n8n-nodes-base.emailSend` (Email Send). Alerts staff via email when a Clio matter creation failure occurs.
  - **Configuration:** Maps error summary text to email body content.
  - **Key Expressions:** `toEmail: {{ $vars.FIRM_EMAIL }}`
  - **Connections:** Input from `Build Error Response Payload`; has no downstream outputs.
  - **Credentials:** `Email account (SMTP)` (SMTP).
  - **Version Requirements:** Version 2.

- **Respond Error To Intake Form**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook). Returns an HTTP 502 status code and user-friendly error message back to the webhook caller.
  - **Configuration:** Response code set to `502`, responding with JSON.
  - **Connections:** Input from `Build Error Response Payload`; has no downstream outputs.
  - **Version Requirements:** Version 1.1.

- **When Workflow Error Occurs**
  - **Type & Technical Role:** `n8n-nodes-base.errorTrigger` (Error Trigger). Acts as a global safety net, catching any unhandled exceptions across the entire workflow.
  - **Configuration:** Default error trigger settings.
  - **Connections:** Output connects to `Notify Staff Of Workflow Error`.
  - **Version Requirements:** Version 1.

- **Notify Staff Of Workflow Error**
  - **Type & Technical Role:** `n8n-nodes-base.emailSend` (Email Send). Sends an alert email containing the specific exception message when an unhandled error occurs.
  - **Configuration:** Plain text email mapping `$json.execution.error.message`.
  - **Key Expressions:** `={{ 'Unhandled error in the Clio Intake Sync workflow: ' + $json.execution.error.message }}`
  - **Connections:** Input from `When Workflow Error Occurs`; has no downstream outputs.
  - **Credentials:** `Email account (SMTP)` (SMTP).
  - **Version Requirements:** Version 2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Clio Intake Sync<br><br>### How it works<br><br>1. The workflow triggers when a new intake form is submitted and validates the intake data.<br>2. It searches for existing contacts in Clio and either selects the existing contact or creates a new one.<br>3. Creates a matter in Clio linked to the contact and handles success or failure outcomes.<br>4. On successful matter creation, it notifies staff via email and optionally Slack, and also sends a client confirmation email after compliance checks.<br>5. If errors occur during processing or workflow execution, it notifies staff accordingly.<br><br>### Setup steps<br><br>- [ ] Configure the webhook trigger to receive intake form submissions.<br>- [ ] Set up Clio API credentials and base URL in workflow variables.<br>- [ ] Provide Slack webhook URL if Slack notifications are desired.<br>- [ ] Configure email credentials for sending notifications and confirmations.<br>- [ ] Ensure the compliance guardrail sub-workflow is available and correctly referenced.<br><br>### Customization<br><br>Customize the email notification templates and Slack message content in the respective code nodes. Adjust the compliance workflow and branching logic as needed for firm policies. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Trigger and intake validation<br><br>Handles the reception of new intake form data via webhook and validates and normalizes the payload. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Clio contact lookup and decision<br><br>Searches Clio for existing contacts and determines if a matching contact exists, branching based on presence. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Existing contact handling<br><br>Processes and prepares the existing contact ID for further matter creation. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## New contact creation<br><br>Creates a new contact in Clio and extracts the new contact ID for use in matter creation. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Matter creation and response<br><br>Creates a new matter in Clio, then routes workflow based on success or failure of the creation. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Staff notification on success<br><br>Builds and sends project manager notification emails and optionally sends Slack alerts about the new intake. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Client confirmation message flow<br><br>Builds client confirmation messages, invokes compliance checks, and sends confirmation emails or skips based on compliance. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Error notification on failure<br><br>Builds error payloads and notifies staff of Clio failures, then responds to the form with errors. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Unhandled workflow error handling<br><br>Trigger for any unhandled workflow errors and notification to staff of such failures. |
| When New Client Intake Form Submitted | n8n-nodes-base.webhook | Entry point — point your Fluent Forms webhook feed at this URL. Expected form fields (adjust names in the next node if your form labels differ):<br>  full_name, email, phone, matter_type, case_details, referral_source (optional)<br><br>Activate the workflow, then open this node and copy the Production URL from the panel on the right. | None | Validate And Normalize Intake Data | |
| Validate And Normalize Intake Data | n8n-nodes-base.code | Required fields: full_name, email, matter_type. Phone and case_details are optional but recommended.<br>Throws a descriptive error (caught downstream by the Error Trigger) if a required field is missing, so a bad form submission never silently creates a broken Clio record. | When New Client Intake Form Submitted | Fetch Existing Clio Contact | |
| Fetch Existing Clio Contact | n8n-nodes-base.httpRequest | Dedupe check — searches Clio contacts by email before creating a new one.<br>Set CLIO_BASE_URL in project → Variables (https://app.clio.com, or https://eu.app.clio.com for EU firms).<br>Credential: Header Auth named 'Clio API (Bearer token)' — in Clio go to Settings → API Keys → New API key, then in n8n set the header Name to 'Authorization' and Value to 'Bearer YOUR_TOKEN'.<br>If this call errors (e.g. Clio rate limit), the workflow continues and treats it as 'not found' rather than blocking the whole intake — a duplicate contact is a smaller problem than a lost lead. | Validate And Normalize Intake Data | Determine If Contact Exists | |
| Determine If Contact Exists | n8n-nodes-base.code | Runs safely even if the Clio search above errored or returned zero rows — 'no data' is treated as 'contact does not exist'. | Fetch Existing Clio Contact | If Contact Exists In Clio | |
| If Contact Exists In Clio | n8n-nodes-base.if | Yes → reuse existing_contact_id, skip creation.<br>No → create a new Clio contact from the intake fields. | Determine If Contact Exists | Extract Existing Contact Id,<br>Create New Clio Contact | |
| Extract Existing Contact Id | n8n-nodes-base.code | Contact already exists in Clio — no duplicate is created. Forwards existing_contact_id as contact_id for the Matter step. | If Contact Exists In Clio | Create New Matter In Clio | |
| Create New Clio Contact | n8n-nodes-base.httpRequest | Creates a new Person contact in Clio. If this call fails, execution routes to the error-notification branch below rather than silently dropping the lead. | If Contact Exists In Clio | Extract New Contact Id | |
| Extract New Contact Id | n8n-nodes-base.code | Throws loudly if Clio didn't return a usable contact id — this stops the run and surfaces in the Error Trigger rather than creating a Matter with no linked contact. | Create New Clio Contact | Create New Matter In Clio | |
| Create New Matter In Clio | n8n-nodes-base.httpRequest | Both the 'existing contact' and 'new contact' branches above feed into this node — only one branch fires per run, so no Merge node is needed. Creates the Matter linked to contact_id, with practice_area mapped from the form's matter_type. | Extract Existing Contact Id,<br>Extract New Contact Id | If Matter Creation Successful | |
| If Matter Creation Successful | n8n-nodes-base.if | Yes → matter has a real Clio object, proceed to staff notification.<br>No → Clio rejected the request (e.g. unknown practice area name) — route to error notification instead of pretending success. | Create New Matter In Clio | Build PM Notification Message,<br>Respond Success To Intake Form,<br>Build Error Response Payload | |
| Build PM Notification Message | n8n-nodes-base.code | Builds a plain routing summary for staff — name, contact info, practice area, matter link. Deliberately contains no legal assessment or screening language (ABA Op. 512 — operational only). | If Matter Creation Successful | Send Staff Email Notification,<br>Build Client Confirmation Message | |
| Send Staff Email Notification | n8n-nodes-base.emailSend | Emails the PM/staff a plain routing summary the moment a Matter is created — no legal opinion, just who, what, and a link to the record.<br><br>Credential: project → Credentials → New → SMTP, saved as 'Email account (SMTP)'. Works with Gmail app passwords, Outlook 365, Zoho, or any SMTP provider.<br><br>Set FIRM_FROM_EMAIL and FIRM_EMAIL in project → Variables (can be the same address). | Build PM Notification Message | If Slack Webhook Configured | |
| If Slack Webhook Configured | n8n-nodes-base.if | Checks whether the optional Slack alert is configured.<br><br>Yes (FIRM_SLACK_WEBHOOK_URL is set) → also post the intake summary to Slack.<br>No (variable is blank) → skip silently. The staff email alert has already been sent regardless.<br><br>To enable: Slack → your workspace → Apps → Incoming Webhooks → Add New Webhook → choose a channel → copy the URL → paste into FIRM_SLACK_WEBHOOK_URL in project → Variables. | Send Staff Email Notification | Post Slack Alert To Staff,<br>Skip Slack Notification | |
| Post Slack Alert To Staff | n8n-nodes-base.httpRequest | Posts the same routing summary to Slack via your Incoming Webhook URL.<br><br>Setup: Slack → your workspace → Apps → Incoming Webhooks → Add New Webhook → choose a channel → copy the URL → paste into FIRM_SLACK_WEBHOOK_URL in project → Variables.<br><br>If Slack returns an error (e.g. the webhook was revoked), the workflow continues — the staff already received the email alert. | If Slack Webhook Configured | None | |
| Skip Slack Notification | n8n-nodes-base.noOp | FIRM_SLACK_WEBHOOK_URL is not set — Slack alert skipped. The staff email alert has already been sent. Add FIRM_SLACK_WEBHOOK_URL to project → Variables to enable Slack pings. | If Slack Webhook Configured | None | |
| Build Client Confirmation Message | n8n-nodes-base.code | Purely operational acknowledgement — no legal advice, no case assessment. To send by SMS instead, set channel: 'sms' and recipient: d.phone here. | Build PM Notification Message | Execute Bar Compliance Check | |
| Execute Bar Compliance Check | n8n-nodes-base.executeWorkflow | Calls the shared Bar-Compliance Guardrail workflow (OPS1) — checks opt-out status and appends the required disclaimer before any client-facing message goes out.<br><br>Set GUARDRAIL_WORKFLOW_ID in project → Variables to the numeric ID from the guardrail workflow's URL: .../workflow/WORKFLOW_ID.<br><br>Passes: channel, recipient, message, template_id ('PAC-20'), matter_id.<br>Returns: approved (bool), message_out (disclaimer-appended text), suppression_reason.<br><br>Prerequisite: Bar-Compliance Guardrail must be deployed and active. | Build Client Confirmation Message | If Compliance Approved | |
| If Compliance Approved | n8n-nodes-base.if | Yes → send the confirmation using message_out (carries the required disclaimer).<br>No → recipient is opted out or compliance check failed — do not send, no error raised. | Execute Bar Compliance Check | Send Confirmation Email,<br>Skip Confirmation Email | |
| Send Confirmation Email | n8n-nodes-base.emailSend | Credential: 'Email account (SMTP)' — same credential used for the staff alert above. Sends message_out, never the raw draft — that's the version with the opt-out disclaimer attached. | If Compliance Approved | None | |
| Skip Confirmation Email | n8n-nodes-base.noOp | Recipient is opted out (or the compliance check errored) — no confirmation sent. Visible in execution history for audit. | If Compliance Approved | None | |
| Respond Success To Intake Form | n8n-nodes-base.respondToWebhook | Returns a small JSON success payload to Fluent Forms' webhook feed — useful if you show a custom confirmation state on the form. | If Matter Creation Successful | None | |
| Build Error Response Payload | n8n-nodes-base.code | Only reached when Clio rejects the Matter creation request (e.g. bad practice_area name, expired token). | If Matter Creation Successful | Send Clio Failure Notification,<br>Respond Error To Intake Form | |
| Send Clio Failure Notification | n8n-nodes-base.emailSend | Alerts staff by email when Clio rejects the Matter creation request, so the lead still gets a human follow-up. | Build Error Response Payload | None | |
| Respond Error To Intake Form | n8n-nodes-base.respondToWebhook | Returned to Fluent Forms if Clio matter creation failed — still a graceful message to the client, not a raw error. | Build Error Response Payload | None | |
| When Workflow Error Occurs | n8n-nodes-base.errorTrigger | Catches any unhandled exception anywhere in this workflow (e.g. the validation node's thrown errors) and emails staff — a safety net on top of the explicit error branch. | None | Notify Staff Of Workflow Error | |
| Notify Staff Of Workflow Error | n8n-nodes-base.emailSend | Safety-net alert — fires for any error not already caught by the explicit Clio-failure branch. | When Workflow Error Occurs | None | |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create the Webhook Trigger
1. Add a **Webhook** node named `When New Client Intake Form Submitted`.
2. Set **HTTP Method** to `POST`.
3. Set **Path** to `client-intake`.
4. Set **Response Mode** to `Respond Using 'Respond to Webhook' Node`.

#### Step 2: Add Intake Validation
1. Add a **Code** node named `Validate And Normalize Intake Data`.
2. Connect `When New Client Intake Form Submitted` to this node.
3. Paste JavaScript logic to validate mandatory fields (`full_name`, `email`, `matter_type`), normalize phone numbers to E.164 format, and standardize practice areas.

#### Step 3: Implement Clio Contact Lookup
1. Add an **HTTP Request** node named `Fetch Existing Clio Contact`.
2. Connect `Validate And Normalize Intake Data` to this node.
3. Configure authentication using **Header Auth** (`Clio API (Bearer token)`).
4. Set URL to `={{ $vars.CLIO_BASE_URL }}/api/v4/contacts.json?query={{ $json.email }}&fields=id,name,primary_email_address`.
5. Set **On Error** to `Continue Regular Output`.

#### Step 4: Add Contact Decision Logic
1. Add a **Code** node named `Determine If Contact Exists`.
2. Connect `Fetch Existing Clio Contact` to this node.
3. Add an **If** node named `If Contact Exists In Clio`.
4. Configure condition: `{{ $json.contact_exists }}` equals `true`.

#### Step 5: Handle Existing and New Contacts
1. Add a **Code** node named `Extract Existing Contact Id` connected to the True branch of `If Contact Exists In Clio`.
2. Add an **HTTP Request** node named `Create New Clio Contact` connected to the False branch of `If Contact Exists In Clio`:
   - Method: `POST`
   - URL: `={{ $vars.CLIO_BASE_URL }}/api/v4/contacts.json`
   - Body: JSON payload specifying Person name, email addresses, and phone numbers.
   - Auth: Header Auth (`Clio API (Bearer token)`).
3. Add a **Code** node named `Extract New Contact Id` connected after `Create New Clio Contact` to validate and extract the created record ID.

#### Step 6: Create Clio Matter
1. Add an **HTTP Request** node named `Create New Matter In Clio`.
2. Connect both `Extract Existing Contact Id` and `Extract New Contact Id` to this node.
3. Set Method to `POST`, URL to `={{ $vars.CLIO_BASE_URL }}/api/v4/matters.json`.
4. Provide JSON body mapping `client.id`, `practice_area.name`, `description`, and `status: 'Open'`.
5. Add an **If** node named `If Matter Creation Successful` to evaluate if `{{ $json.data }}` is not empty.

#### Step 7: Configure Success and Notification Branch
1. Add a **Code** node named `Build PM Notification Message` connected to the True branch of `If Matter Creation Successful`.
2. Add an **Email Send** node named `Send Staff Email Notification`:
   - Credential: SMTP (`Email account (SMTP)`)
   - To: `={{ $vars.FIRM_EMAIL }}`
   - From: `={{ $vars.FIRM_FROM_EMAIL }}`
3. Add an **If** node named `If Slack Webhook Configured` checking if `{{ $vars.FIRM_SLACK_WEBHOOK_URL }}` is not empty.
4. Add an **HTTP Request** node named `Post Slack Alert To Staff` connected to the True branch of Slack check, posting Block Kit payloads to `={{ $vars.FIRM_SLACK_WEBHOOK_URL }}`.
5. Add a **No Operation** node named `Skip Slack Notification` connected to the False branch.
6. Add a **Respond to Webhook** node named `Respond Success To Intake Form` connected to the True branch of `If Matter Creation Successful`, returning JSON status success.

#### Step 8: Configure Client Confirmation & Compliance Guardrail
1. Add a **Code** node named `Build Client Confirmation Message` connected to `Build PM Notification Message`.
2. Add an **Execute Workflow** node named `Execute Bar Compliance Check`:
   - Workflow ID: `={{ $vars.GUARDRAIL_WORKFLOW_ID }}`
   - Input mapping: Pass `channel`, `recipient`, `message`, `matter_id`, and `template_id`.
3. Add an **If** node named `If Compliance Approved` checking `={{ $json.approved }}`.
4. Add an **Email Send** node named `Send Confirmation Email` connected to the True branch, mapping `={{ $json.message_out }}` to the email body.
5. Add a **No Operation** node named `Skip Confirmation Email` connected to the False branch.

#### Step 9: Configure Failure and Error Handling
1. Add a **Code** node named `Build Error Response Payload` connected to the False branch of `If Matter Creation Successful`.
2. Add an **Email Send** node named `Send Clio Failure Notification` alerting staff via `{{ $vars.FIRM_EMAIL }}`.
3. Add a **Respond to Webhook** node named `Respond Error To Intake Form` returning HTTP status `502` and a user-friendly error message.
4. Add an **Error Trigger** node named `When Workflow Error Occurs`.
5. Connect an **Email Send** node named `Notify Staff Of Workflow Error` to email staff with `={{ 'Unhandled error in the Clio Intake Sync workflow: ' + $json.execution.error.message }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Clio API Integration Reference | Configure environment variable `CLIO_BASE_URL` (e.g., `https://app.clio.com` or `https://eu.app.clio.com`). |
| Staff and Sender Email Variables | Define `FIRM_EMAIL` and `FIRM_FROM_EMAIL` in n8n workflow variables. |
| Slack Notification Setup | Optional: Define `FIRM_SLACK_WEBHOOK_URL` using Slack Incoming Webhooks. |
| Compliance Guardrail Sub-Workflow | Prerequisite: Deploy the Bar-Compliance Guardrail workflow and set its numeric ID in `GUARDRAIL_WORKFLOW_ID`. |