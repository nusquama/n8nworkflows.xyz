Send missed call text-backs with Twilio and email dispatcher approvals

https://n8nworkflows.xyz/workflows/send-missed-call-text-backs-with-twilio-and-email-dispatcher-approvals-20319


# Send missed call text-backs with Twilio and email dispatcher approvals

### 1. Workflow Overview

This workflow automates the handling of missed-call webhook events by validating caller information, looking up contacts, evaluating business hours, requesting after-hours dispatcher approval, dispatching SMS text-backs via Twilio, and logging all outcomes while alerting dispatchers via email.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming missed-call payloads, initializes configuration settings, normalizes the phone number, and determines after-hours status.
- **1.2 Validation & Fallback Routing:** Validates the phone number; valid numbers proceed to processing, while invalid numbers trigger an HTTP 400 response, a data table log entry, and a notification email.
- **1.3 Contact Lookup & Business Hours Evaluation:** Acknowledges valid calls with HTTP 202, queries an n8n Data Table for contact details, and evaluates whether the call occurred outside business hours.
- **1.4 After-Hours Approval Gate:** Sends an approval email to the dispatcher if the call occurred outside business hours. If approved, processing continues; if declined, the event is logged as unsent.
- **1.5 Message Preparation & Dispatch:** Classifies callers as known contacts or first-time callers, formats the respective SMS message, and sends it via Twilio.
- **1.6 Outcome Logging & Dispatcher Notification:** Handles success and failure branches following SMS delivery attempts, recording outcomes in an n8n Data Table and sending confirmation or failure-alert emails to the dispatcher.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block captures the incoming missed-call webhook, sets global runtime parameters, and parses/normalizes raw incoming data into a standardized format.

- **When Call Missed**
  - **Type and Technical Role:** `n8n-nodes-base.webhook` (Trigger)
  - **Configuration Choices:** Listens for HTTP `POST` requests on path `demo-missed-call` using response mode `responseNode`.
  - **Key Expressions or Variables:** None (trigger node).
  - **Input / Output Connections:** Input: External webhook; Output: `Prepare Settings Data`.
  - **Version-Specific Requirements:** Type version 2.1.
  - **Edge Cases / Potential Failures:** Network timeouts or improper webhook configuration on the telecom provider side.

- **Prepare Settings Data**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Transform)
  - **Configuration Choices:** Assigns static configuration variables including business name, Twilio sender number, schedule URL, dispatcher/sender email addresses, timezone, and business operating hours (7 AM to 7 PM).
  - **Key Expressions or Variables:** Sets string, number, and boolean configuration fields.
  - **Input / Output Connections:** Input: `When Call Missed`; Output: `Normalize Call Data`.
  - **Version-Specific Requirements:** Type version 3.4.
  - **Edge Cases / Potential Failures:** Ensure email addresses and timezone string match valid formats.

- **Normalize Call Data**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Transform)
  - **Configuration Choices:** Normalizes incoming phone numbers to E.164 format, captures raw values, assigns ISO timestamps, and evaluates whether the call occurred after hours based on simulation flags or local time zones.
  - **Key Expressions or Variables:** 
    - `phone`: `={{ (() => { const d = String($json.body?.from ?? $json.from ?? "").replace(/\D/g, ""); if (d.length === 10) return "+1" + d; if (d.length === 11 && d.startsWith("1")) return "+" + d; return ""; })() }}`
    - `after_hours`: Evaluates simulation override or compares current hour against `open_hour` and `close_hour`.
  - **Input / Output Connections:** Input: `Prepare Settings Data`; Output: `Check Valid Number`.
  - **Version-Specific Requirements:** Type version 3.4.
  - **Edge Cases / Potential Failures:** Malformed phone payloads resulting in empty normalized strings.

#### 2.2 Validation & Fallback Routing
This block evaluates the normalized phone number, separating valid submissions from malformed payloads.

- **Check Valid Number**
  - **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control)
  - **Configuration Choices:** Checks whether the normalized `phone` field is not empty.
  - **Key Expressions or Variables:** `={{ $json.phone }}` (operator: `notEmpty`).
  - **Input / Output Connections:** Input: `Normalize Call Data`; Output (True): `Acknowledge Valid Call`; Output (False): `Reject Invalid Call`.
  - **Version-Specific Requirements:** Type version 2.3.
  - **Edge Cases / Potential Failures:** Edge cases where numbers contain non-standard country codes not caught by normalization.

- **Reject Invalid Call**
  - **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Response)
  - **Configuration Choices:** Returns an HTTP 400 response with a JSON error payload detailing the validation failure.
  - **Key Expressions or Variables:** `={{ { "error": "validation_error", "message": "Field \"from\" must be a 10-digit US phone number.", "received": $json.raw_from } }}`
  - **Input / Output Connections:** Input: `Check Valid Number` (False); Output: `Record Rejected Calls`.
  - **Version-Specific Requirements:** Type version 1.5.

- **Record Rejected Calls**
  - **Type and Technical Role:** `n8n-nodes-base.dataTable` (Database Integration)
  - **Configuration Choices:** Records rejected events into the `demo_missed_calls` table with status `rejected`.
  - **Key Expressions or Variables:** Maps `phone` to `raw_from`, status to `rejected`, and adds descriptive details.
  - **Input / Output Connections:** Input: `Reject Invalid Call`; Output: `Notify Dispatcher of Bad Call`.
  - **Version-Specific Requirements:** Type version 1.1.

- **Notify Dispatcher of Bad Call**
  - **Type and Technical Role:** `n8n-nodes-base.emailSend` (Notification)
  - **Configuration Choices:** Sends a plain-text email alert to the dispatcher regarding the malformed event. Configured with retry logic (up to 3 tries, 5s delay).
  - **Key Expressions or Variables:** Uses node references pointing back to `Normalize Call Data` for raw values and timestamps.
  - **Input / Output Connections:** Input: `Record Rejected Calls`; Output: None (terminal branch).
  - **Version-Specific Requirements:** Type version 2.1.
  - **Edge Cases / Potential Failures:** SMTP authentication errors or network timeouts.

#### 2.3 Contact Lookup & Business Hours Evaluation
This block acknowledges valid webhook requests, queries the contact database, and checks whether the interaction occurred outside business operating hours.

- **Acknowledge Valid Call**
  - **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Response)
  - **Configuration Choices:** Responds immediately with HTTP 202 and an accepted status JSON body.
  - **Key Expressions or Variables:** `={{ { "status": "accepted", "phone": $json.phone } }}`
  - **Input / Output Connections:** Input: `Check Valid Number` (True); Output: `Search Contact Database`.
  - **Version-Specific Requirements:** Type version 1.5.

- **Search Contact Database**
  - **Type and Technical Role:** `n8n-nodes-base.dataTable` (Database Integration)
  - **Configuration Choices:** Queries the `demo_contacts` data table filtering by phone number. Configured with `continueRegularOutput` on error and always outputs data.
  - **Key Expressions or Variables:** `={{ $('Normalize Call Data').item.json.phone }}`
  - **Input / Output Connections:** Input: `Acknowledge Valid Call`; Output: `Check After Hours`.
  - **Version-Specific Requirements:** Type version 1.1.

- **Check After Hours**
  - **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control)
  - **Configuration Choices:** Evaluates the `after_hours` boolean flag.
  - **Key Expressions or Variables:** `={{ $('Normalize Call Data').item.json.after_hours }}` (operator: `true`).
  - **Input / Output Connections:** Input: `Search Contact Database`; Output (True): `Email Dispatcher for Approval`; Output (False): `Identify Known Customer`.
  - **Version-Specific Requirements:** Type version 2.3.

#### 2.4 After-Hours Approval Gate
This block manages dispatcher intervention for after-hours calls to prevent unwanted middle-of-the-night messaging.

- **Email Dispatcher for Approval**
  - **Type and Technical Role:** `n8n-nodes-base.emailSend` (Approval Workflow)
  - **Configuration Choices:** Uses `sendAndWait` operation with double approval options ("Send the text now" vs "Do not send").
  - **Key Expressions or Variables:** Dynamically constructs email body and subject referencing caller phone, contact name (or first-time caller), and timestamps.
  - **Input / Output Connections:** Input: `Check After Hours` (True); Output: `Check Dispatcher Approval`.
  - **Version-Specific Requirements:** Type version 2.1.
  - **Edge Cases / Potential Failures:** Approval workflow timeouts or email delivery failures.

- **Check Dispatcher Approval**
  - **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control)
  - **Configuration Choices:** Checks if the dispatcher approved the action.
  - **Key Expressions or Variables:** `={{ $json.data.approved }}` (operator: `true`).
  - **Input / Output Connections:** Input: `Email Dispatcher for Approval`; Output (True): `Identify Known Customer`; Output (False): `Record Unsent Messages`.
  - **Version-Specific Requirements:** Type version 2.3.

- **Record Unsent Messages**
  - **Type and Technical Role:** `n8n-nodes-base.dataTable` (Database Integration)
  - **Configuration Choices:** Logs declined after-hours calls to the `demo_missed_calls` table with status `not_sent`.
  - **Key Expressions or Variables:** Extracts caller phone, caller type, and timestamp.
  - **Input / Output Connections:** Input: `Check Dispatcher Approval` (False); Output: None (terminal branch).
  - **Version-Specific Requirements:** Type version 1.1.

#### 2.5 Message Preparation & Dispatch
This block classifies the caller type, prepares customized SMS content, and sends the message via Twilio.

- **Identify Known Customer**
  - **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control)
  - **Configuration Choices:** Checks if a contact name exists from the database search.
  - **Key Expressions or Variables:** `={{ $('Search Contact Database').item.json.name ?? '' }}` (operator: `notEmpty`).
  - **Input / Output Connections:** Input: `Check After Hours` (False) or `Check Dispatcher Approval` (True); Output (True): `Prepare Message for Known Caller`; Output (False): `Prepare Message for New Caller`.
  - **Version-Specific Requirements:** Type version 2.3.

- **Prepare Message for Known Caller**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Transform)
  - **Configuration Choices:** Builds a personalized SMS template using the contact's first name, business name, and scheduling URL.
  - **Key Expressions or Variables:** `={{ 'Hi ' + $('Search Contact Database').item.json.name.split(' ')[0] + ', this is ...' }}`
  - **Input / Output Connections:** Input: `Identify Known Customer` (True); Output: `Send SMS Response`.
  - **Version-Specific Requirements:** Type version 3.4.

- **Prepare Message for New Caller**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Transform)
  - **Configuration Choices:** Builds a generic greeting SMS template for first-time callers.
  - **Key Expressions or Variables:** Uses business name and schedule URL from settings data.
  - **Input / Output Connections:** Input: `Identify Known Customer` (False); Output: `Send SMS Response`.
  - **Version-Specific Requirements:** Type version 3.4.

- **Send SMS Response**
  - **Type and Technical Role:** `n8n-nodes-base.twilio` (Integration)
  - **Configuration Choices:** Sends the SMS via Twilio. Configured with retry logic (`retryOnFail: true`, 2 max tries, 5000ms wait between tries) and routes errors to the error output (`continueErrorOutput`).
  - **Key Expressions or Variables:** 
    - `to`: `={{ $('Normalize Call Data').item.json.phone }}`
    - `from`: `={{ $('Prepare Settings Data').item.json.sms_from }}`
    - `message`: `={{ $json.message }}`
  - **Input / Output Connections:** Input: `Prepare Message for Known Caller` or `Prepare Message for New Caller`; Output (Main / Success): `Record Sent Messages`; Output (Error / Failure): `Record Failed Texts`.
  - **Version-Specific Requirements:** Type version 1.

#### 2.6 Outcome Logging & Dispatcher Notification
This block records successful or failed delivery outcomes and notifies the dispatcher accordingly.

- **Record Sent Messages**
  - **Type and Technical Role:** `n8n-nodes-base.dataTable` (Database Integration)
  - **Configuration Choices:** Records successful SMS dispatches into the `demo_missed_calls` table with status `sent`.
  - **Key Expressions or Variables:** `={{ 'SMS ' + ($json.sid ?? '') + ' ' + ($json.status ?? '') }}`
  - **Input / Output Connections:** Input: `Send SMS Response` (Success); Output: `Email Dispatcher: Text Sent`.
  - **Version-Specific Requirements:** Type version 1.1.

- **Email Dispatcher: Text Sent**
  - **Type and Technical Role:** `n8n-nodes-base.emailSend` (Notification)
  - **Configuration Choices:** Sends a confirmation email to the dispatcher. Configured with retry logic (3 max tries, 5s delay).
  - **Key Expressions or Variables:** References node data for phone numbers, caller names, timestamps, and Twilio status.
  - **Input / Output Connections:** Input: `Record Sent Messages`; Output: None (terminal branch).
  - **Version-Specific Requirements:** Type version 2.1.

- **Record Failed Texts**
  - **Type and Technical Role:** `n8n-nodes-base.dataTable` (Database Integration)
  - **Configuration Choices:** Logs failed SMS delivery attempts into the `demo_missed_calls` table with status `failed`.
  - **Key Expressions or Variables:** Extracts error messages from Twilio failure outputs.
  - **Input / Output Connections:** Input: `Send SMS Response` (Error); Output: `Email Dispatcher: Call Back Needed`.
  - **Version-Specific Requirements:** Type version 1.1.

- **Email Dispatcher: Call Back Needed**
  - **Type and Technical Role:** `n8n-nodes-base.emailSend` (Notification)
  - **Configuration Choices:** Sends an action-required alert email to the dispatcher instructing them to manually call back the missed caller. Configured with retry logic (3 max tries, 5s delay).
  - **Key Expressions or Variables:** References caller details and timestamps.
  - **Input / Output Connections:** Input: `Record Failed Texts`; Output: None (terminal branch).
  - **Version-Specific Requirements:** Type version 2.1.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Documentation and setup guide | None | None | Demo: missed call text-back (fictional data)<br><br>Demonstration workflow, fictional data. Built to show how these are put together. It is not a client system, and no client data appears in it.<br><br>### How it works<br><br>This workflow receives a missed-call webhook, normalizes the caller information, and validates that the phone number is usable. Valid calls are acknowledged, checked against a contact table, routed through after-hours dispatcher approval when needed, then sent an appropriate SMS text-back for known or first-time callers. It logs successful sends, failures, rejected events, and cases where a dispatcher did not approve the text, while notifying dispatchers by email when attention is needed.<br><br>### Setup steps<br><br>- Configure the webhook URL with the phone system or call-tracking provider that emits missed-call events.<br>- Update the Prepare Settings Data node with the business name, Twilio/SMS sender number, scheduling URL, and dispatcher email address.<br>- Connect Twilio credentials for the SMS sending node and verify the sender number is approved for outbound messages.<br>- Configure email credentials for dispatcher approval, sent notifications, failure alerts, and malformed-event alerts.<br>- Create two n8n data tables: `demo_contacts` (phone, name) for the contact lookup, and `demo_missed_calls` (phone, caller_type, status, detail, called_at) for logging rejected, skipped, sent, and failed text events.<br>- Review the IF conditions for phone-number validation, business-hours detection, known-customer matching, and dispatcher approval behavior.<br><br>### Testing<br><br>Send a `POST` with `{ "from": "(555) 555-0101" }`. To try the in-hours or after-hours path at any time of day, add `"simulate_after_hours": false` or `true` to the body.<br><br>### Customization<br><br>Customize the known-customer and first-time-caller message templates, business-hours rules, dispatcher approval criteria, and notification recipients to match the real business process. |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Section header and description | None | None | ## Receive and validate call<br><br>Captures the missed-call webhook, applies workflow settings, normalizes incoming call fields, and checks whether the caller number is valid before branching. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Section header and description | None | None | ## Acknowledge and check contact<br><br>Handles the valid-event path by responding to the webhook, looking up the caller in the contact table, and deciding whether the call is outside business hours. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Section header and description | None | None | ## Handle malformed event<br><br>Handles invalid phone-number events by returning a rejection response, logging the rejected event, and emailing the dispatcher about the bad payload. |
| **Sticky Note4** | `n8n-nodes-base.stickyNote` | Section header and description | None | None | ## After-hours approval gate<br><br>Requests dispatcher approval for after-hours text-backs, evaluates whether approval was granted, and logs cases where no message should be sent. |
| **Sticky Note5** | `n8n-nodes-base.stickyNote` | Section header and description | None | None | ## Prepare and send message<br><br>Classifies the caller as known or first-time, builds the appropriate SMS message, and sends the text-back through Twilio. |
| **Sticky Note6** | `n8n-nodes-base.stickyNote` | Section header and description | None | None | ## Log successful text<br><br>Records successful SMS sends and notifies the dispatcher that the text-back was delivered or accepted for sending. |
| **Sticky Note7** | `n8n-nodes-base.stickyNote` | Section header and description | None | None | ## Handle text failure<br><br>Records failed SMS attempts and alerts the dispatcher to manually call the missed caller back. |
| **When Call Missed** | `n8n-nodes-base.webhook` | Webhook trigger for missed calls | External Webhook | Prepare Settings Data | Receive and validate call |
| **Prepare Settings Data** | `n8n-nodes-base.set` | Define global configuration variables | When Call Missed | Normalize Call Data | Receive and validate call |
| **Normalize Call Data** | `n8n-nodes-base.set` | Normalize phone number and timestamp | Prepare Settings Data | Check Valid Number | Receive and validate call |
| **Check Valid Number** | `n8n-nodes-base.if` | Validate phone number presence | Normalize Call Data | Acknowledge Valid Call, Reject Invalid Call | Receive and validate call |
| **Acknowledge Valid Call** | `n8n-nodes-base.respondToWebhook` | Send HTTP 202 response | Check Valid Number | Search Contact Database | Acknowledge and check contact |
| **Search Contact Database** | `n8n-nodes-base.dataTable` | Look up contact by phone | Acknowledge Valid Call | Check After Hours | Acknowledge and check contact |
| **Check After Hours** | `n8n-nodes-base.if` | Evaluate business hours status | Search Contact Database | Email Dispatcher for Approval, Identify Known Customer | Acknowledge and check contact |
| **Email Dispatcher for Approval** | `n8n-nodes-base.emailSend` | Send approval request to dispatcher | Check After Hours | Check Dispatcher Approval | After-hours approval gate |
| **Check Dispatcher Approval** | `n8n-nodes-base.if` | Evaluate dispatcher decision | Email Dispatcher for Approval | Identify Known Customer, Record Unsent Messages | After-hours approval gate |
| **Record Unsent Messages** | `n8n-nodes-base.dataTable` | Log declined after-hours calls | Check Dispatcher Approval | None | After-hours approval gate |
| **Identify Known Customer** | `n8n-nodes-base.if` | Check if contact name exists | Check After Hours / Check Dispatcher Approval | Prepare Message for Known Caller, Prepare Message for New Caller | Prepare and send message |
| **Prepare Message for Known Caller** | `n8n-nodes-base.set` | Build personalized SMS template | Identify Known Customer | Send SMS Response | Prepare and send message |
| **Prepare Message for New Caller** | `n8n-nodes-base.set` | Build standard greeting SMS template | Identify Known Customer | Send SMS Response | Prepare and send message |
| **Send SMS Response** | `n8n-nodes-base.twilio` | Send SMS via Twilio | Prepare Message for Known Caller / Prepare Message for New Caller | Record Sent Messages, Record Failed Texts | Prepare and send message |
| **Reject Invalid Call** | `n8n-nodes-base.respondToWebhook` | Send HTTP 400 validation error | Check Valid Number | Record Rejected Calls | Handle malformed event |
| **Record Rejected Calls** | `n8n-nodes-base.dataTable` | Log rejected malformed payloads | Reject Invalid Call | Notify Dispatcher of Bad Call | Handle malformed event |
| **Notify Dispatcher of Bad Call** | `n8n-nodes-base.emailSend` | Email dispatcher about bad call payload | Record Rejected Calls | None | Handle malformed event |
| **Record Sent Messages** | `n8n-nodes-base.dataTable` | Log successful SMS sends | Send SMS Response | Email Dispatcher: Text Sent | Log successful text |
| **Email Dispatcher: Text Sent** | `n8n-nodes-base.emailSend` | Notify dispatcher of successful text | Record Sent Messages | None | Log successful text |
| **Record Failed Texts** | `n8n-nodes-base.dataTable` | Log failed SMS delivery attempts | Send SMS Response | Email Dispatcher: Call Back Needed | Handle text failure |
| **Email Dispatcher: Call Back Needed** | `n8n-nodes-base.emailSend` | Alert dispatcher to call back caller | Record Failed Texts | None | Handle text failure |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow in n8n manually:

1. **Create Data Tables:**
   - Create a Data Table named `demo_contacts` with columns: `phone` (String), `name` (String).
   - Create a Data Table named `demo_missed_calls` with columns: `phone` (String), `caller_type` (String), `status` (String), `detail` (String), `called_at` (String).

2. **Add and Configure the Webhook Trigger:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`).
   - Set **HTTP Method** to `POST` and **Path** to `demo-missed-call`. Set **Response Mode** to `Response Node`.

3. **Add and Configure Settings and Normalization Nodes:**
   - Add a **Set** node named `Prepare Settings Data`. Configure assignments for `business_name`, `sms_from`, `schedule_url`, `dispatcher_email`, `sender_email`, `timezone`, `open_hour`, and `close_hour`.
   - Add a **Set** node named `Normalize Call Data`. Add JavaScript expressions to parse incoming phone numbers, assign ISO timestamps, and evaluate the `after_hours` boolean flag.

4. **Add Validation and Fallback Nodes:**
   - Add an **If** node named `Check Valid Number`. Set condition to verify that `{{ $json.phone }}` is not empty.
   - *Valid Path (True):*
     - Add a **Respond to Webhook** node named `Acknowledge Valid Call` with response code `202` and JSON body.
     - Add a **Data Table** node named `Search Contact Database` (operation: `get`, table: `demo_contacts`, match condition: `phone` equals normalized phone). Enable `continueRegularOutput` and `alwaysOutputData`.
     - Add an **If** node named `Check After Hours` to evaluate if `after_hours` is true.
   - *Invalid Path (False):*
     - Add a **Respond to Webhook** node named `Reject Invalid Call` with response code `400` and validation error JSON.
     - Add a **Data Table** node named `Record Rejected Calls` (operation: `append`, table: `demo_missed_calls`, status: `rejected`).
     - Add an **Email Send** node named `Notify Dispatcher of Bad Call` using SMTP credentials with retry enabled (3 tries, 5s delay).

5. **Add After-Hours Approval Gate:**
   - From the `Check After Hours` True output, add an **Email Send** node named `Email Dispatcher for Approval` configured with `sendAndWait` operation and double approval buttons ("Send the text now" vs "Do not send").
   - Add an **If** node named `Check Dispatcher Approval` to check `{{ $json.data.approved }}`.
   - *False Branch:* Add a **Data Table** node named `Record Unsent Messages` (operation: `append`, table: `demo_missed_calls`, status: `not_sent`).

6. **Add Message Preparation and Dispatch Nodes:**
   - From `Check After Hours` (False) and `Check Dispatcher Approval` (True), route into an **If** node named `Identify Known Customer` checking if contact name is not empty.
   - *True Branch:* Add a **Set** node named `Prepare Message for Known Caller` with personalized messaging templates.
   - *False Branch:* Add a **Set** node named `Prepare Message for New Caller` with generic greeting templates.
   - Route both message set nodes into a **Twilio** node named `Send SMS Response`. Configure credentials, `to`, `from`, and `message` parameters. Enable retry on fail (2 tries, 5s delay) and route error output to handle failures.

7. **Add Success and Failure Logging / Notification Nodes:**
   - *Success Output:*
     - Add a **Data Table** node named `Record Sent Messages` (operation: `append`, table: `demo_missed_calls`, status: `sent`).
     - Add an **Email Send** node named `Email Dispatcher: Text Sent` with retry enabled.
   - *Error Output:*
     - Add a **Data Table** node named `Record Failed Texts` (operation: `append`, table: `demo_missed_calls`, status: `failed`).
     - Add an **Email Send** node named `Email Dispatcher: Call Back Needed` with retry enabled.

8. **Establish Connections:** Connect all nodes as outlined in the block descriptions and summary table.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Demonstration workflow built with fictional data to illustrate missed-call automation patterns. | Workflow Overview |
| Requires active Twilio credentials and configured SMTP email credentials for operational deployment. | Integration Setup |