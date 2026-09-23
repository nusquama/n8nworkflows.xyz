Send engagement letters and open Clio matters with DocuSign and Google Sheets

https://n8nworkflows.xyz/workflows/send-engagement-letters-and-open-clio-matters-with-docusign-and-google-sheets-19678


# Send engagement letters and open Clio matters with DocuSign and Google Sheets

### 1. Workflow Overview

This workflow automates the legal practice lifecycle for issuing engagement letters, securing attorney approval, capturing client e-signatures, and initializing active firm matters. It bridges intake qualification data with CRM contact management, document generation via DocuSign, internal tracking via Google Sheets, and automated matter creation in Clio.

The operational logic is organized into five functional blocks:
- **1.1 Input Reception & Contact Management:** Captures qualified lead webhook data, searches Clio for existing contacts by email, creates new contacts if necessary, and retrieves DocuSign authentication context.
- **1.2 Engagement Letter Envelope Creation:** Builds a pre-filled DocuSign envelope in draft mode based on a firm template, logs the pending review to Google Sheets, and emails an approval link to the assigned attorney.
- **1.3 Attorney Decision Processing:** Receives the attorney's approval webhook, validates the request, verifies the tracking log, and transitions the DocuSign envelope status from draft to sent so the client receives it.
- **1.4 DocuSign Signature Webhook Handling:** Listens for DocuSign Connect execution events, filters out non-completion updates, and isolates fully signed envelopes.
- **1.5 Signature Completion Processing & Matter Opening:** Matches signed envelopes to the pipeline log, automatically opens a new legal matter in Clio, updates the tracking log, and notifies firm staff.

---

### 2. Block-by-Block Analysis

---

### 2.1 Input Reception & Contact Management

#### Overview
This block ingests incoming lead qualification payloads, standardizes lead fields, queries Clio to prevent duplicate contact creation, handles branching logic for existing vs. new records, and retrieves DocuSign API endpoints.

#### Nodes Involved
- `When Lead Qualified For Engagement`
- `Parse Lead Qualification Data`
- `Fetch Existing Clio Contacts`
- `Determine If Clio Contact Exists`
- `If Clio Contact Exists`
- `Create New Clio Contact`
- `Merge New Clio Contact Id`
- `Use Existing Clio Contact Id`
- `Fetch DocuSign User Info`

#### Node Details

##### When Lead Qualified For Engagement
- **Type & Technical Role:** `n8n-nodes-base.webhook` (Webhook Trigger)
- **Configuration:** HTTP Method set to `POST`, listening on path `engagement-letter-intake`.
- **Key Expressions:** None
- **Input / Output Connections:** Input: None (Trigger) | Output: `Parse Lead Qualification Data`
- **Version-Specific Requirements:** v2.1
- **Edge Cases & Failure Types:** Invalid payload format or unauthorized network access. Ensure the calling system sends a valid JSON payload containing contact parameters.

##### Parse Lead Qualification Data
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Extracts and normalizes lead fields (`full_name`, `email`, `phone`, `practice_area`, `case_details`, `attorney_email`) from the incoming request body.
- **Key Expressions:** `{{ $json.body || $json }}`
- **Input / Output Connections:** Input: `When Lead Qualified For Engagement` | Output: `Fetch Existing Clio Contacts`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Missing fields default to empty strings to prevent execution halts.

##### Fetch Existing Clio Contacts
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request)
- **Configuration:** GET request to `={{ $vars.CLIO_BASE_URL }}/api/v4/contacts.json` using generic OAuth2 authentication. Configured with query parameters to filter by lead email.
- **Key Expressions:** `={{ $vars.CLIO_BASE_URL }}/api/v4/contacts.json`, query parameter `query = {{ $json.email }}`
- **Input / Output Connections:** Input: `Parse Lead Qualification Data` | Output: `Determine If Clio Contact Exists`
- **Version-Specific Requirements:** v4.4
- **Edge Cases & Failure Types:** Clio API rate limits or expired OAuth2 tokens. `neverError: true` prevents abrupt crashes on network faults.

##### Determine If Clio Contact Exists
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Iterates through Clio API results to find an exact, case-insensitive email match. Appends `existing_contact_id` to the output dataset.
- **Key Expressions:** Evaluates primary email addresses against incoming lead data.
- **Input / Output Connections:** Input: `Fetch Existing Clio Contacts` | Output: `If Clio Contact Exists`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Empty result arrays return a null identifier.

##### If Clio Contact Exists
- **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Router)
- **Configuration:** Evaluates whether `existing_contact_id` evaluates to a truthy value (`true`).
- **Key Expressions:** `={{ !!$json.existing_contact_id }}`
- **Input / Output Connections:** Input: `Determine If Clio Contact Exists` | Output: `Use Existing Clio Contact Id` (True branch), `Create New Clio Contact` (False branch)
- **Version-Specific Requirements:** v2.3
- **Edge Cases & Failure Types:** None.

##### Create New Clio Contact
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request)
- **Configuration:** POST request to `={{ $vars.CLIO_BASE_URL }}/api/v4/contacts.json` using generic OAuth2. Sends JSON payload with person details.
- **Key Expressions:** Payload maps `full_name`, `email`, and `phone`.
- **Input / Output Connections:** Input: `If Clio Contact Exists` (False) | Output: `Merge New Clio Contact Id`
- **Version-Specific Requirements:** v4.4
- **Edge Cases & Failure Types:** Validation rejections by Clio (e.g., malformed phone numbers).

##### Merge New Clio Contact Id
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Combines the original lead context with the newly generated Clio contact ID from the HTTP response.
- **Key Expressions:** References parent items from `Determine If Clio Contact Exists`.
- **Input / Output Connections:** Input: `Create New Clio Contact` | Output: `Fetch DocuSign User Info`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Scope leakage if referenced item indexes desynchronize.

##### Use Existing Clio Contact Id
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Assigns the resolved `existing_contact_id` to the standard `contact_id` property.
- **Key Expressions:** Maps `existing_contact_id` to `contact_id`.
- **Input / Output Connections:** Input: `If Clio Contact Exists` (True) | Output: `Fetch DocuSign User Info`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** None.

##### Fetch DocuSign User Info
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request)
- **Configuration:** GET request to `={{ $vars.DOCUSIGN_AUTH_BASE_URL }}/oauth/userinfo` using DocuSign OAuth2 credentials.
- **Key Expressions:** `={{ $vars.DOCUSIGN_AUTH_BASE_URL }}/oauth/userinfo`
- **Input / Output Connections:** Input: `Merge New Clio Contact Id` or `Use Existing Clio Contact Id` | Output: `Build DocuSign Envelope Request`
- **Version-Specific Requirements:** v4.4
- **Edge Cases & Failure Types:** Expired DocuSign authorization grants.

---

### 2.2 Engagement Letter Envelope Creation

#### Overview
This block formats template role parameters, generates a draft DocuSign envelope, registers the review task in Google Sheets, and dispatches an approval notification email to the assigned attorney.

#### Nodes Involved
- `Build DocuSign Envelope Request`
- `Post Draft Envelope To DocuSign`
- `Parse DocuSign Envelope Response`
- `Assign Engagement Review Id`
- `Append Engagement Review To Sheets`
- `Build Attorney Review Email`
- `Send Attorney Review Email`

#### Node Details

##### Build DocuSign Envelope Request
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Maps lead data into DocuSign tab labels (`full_name`, `email`, `phone`, `practice_area`, `case_details`, `date`). Forces envelope status to `created` (draft mode).
- **Key Expressions:** Uses environment variables `ENGAGEMENT_LETTER_TEMPLATE_ID` and `ENGAGEMENT_LETTER_ROLE_NAME`.
- **Input / Output Connections:** Input: `Fetch DocuSign User Info` | Output: `Post Draft Envelope To DocuSign`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Mismatched DocuSign template tab labels will fail population during envelope instantiation.

##### Post Draft Envelope To DocuSign
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request)
- **Configuration:** POST request to DocuSign REST API endpoints (`base_uri` + `/restapi/v2.1/accounts/` + `account_id` + `/envelopes`) using OAuth2.
- **Key Expressions:** Dynamic construction of endpoint URLs using account data.
- **Input / Output Connections:** Input: `Build DocuSign Envelope Request` | Output: `Parse DocuSign Envelope Response`
- **Version-Specific Requirements:** v4.4
- **Edge Cases & Failure Types:** Template permission issues or invalid IDs.

##### Parse DocuSign Envelope Response
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Validates the presence of an `envelopeId` in the DocuSign response and structures operational notes.
- **Key Expressions:** Validates response objects for `envelopeId` and absence of `errorCode`.
- **Input / Output Connections:** Input: `Post Draft Envelope To DocuSign` | Output: `Assign Engagement Review Id`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Error codes returned by DocuSign are caught and formatted into notes.

##### Assign Engagement Review Id
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Generates a unique tracking identifier (`ENGL-<timestamp>`) for the engagement process.
- **Key Expressions:** `review_id: 'ENGL-' + Date.now()`
- **Input / Output Connections:** Input: `Parse DocuSign Envelope Response` | Output: `Append Engagement Review To Sheets`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** None.

##### Append Engagement Review To Sheets
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Spreadsheet Integration)
- **Configuration:** Appends a new row to the "Engagement Pipeline" sheet using `ENGAGEMENT_PIPELINE_SHEET_ID`. Populates tracking metadata (`review_id`, `envelope_id`, `status: pending_review`, etc.).
- **Key Expressions:** `={{ $vars.ENGAGEMENT_PIPELINE_SHEET_ID }}`
- **Input / Output Connections:** Input: `Assign Engagement Review Id` | Output: `Build Attorney Review Email`
- **Version-Specific Requirements:** v4
- **Edge Cases & Failure Types:** Sheet column schema mismatch or quota limits.

##### Build Attorney Review Email
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Constructs HTML-formatted email content containing an interactive approval link pointing to the attorney decision webhook.
- **Key Expressions:** Constructs approval URL using `ENGAGEMENT_DECISION_WEBHOOK_URL` and `review_id`.
- **Input / Output Connections:** Input: `Append Engagement Review To Sheets` | Output: `Send Attorney Review Email`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Unescaped strings in lead details could introduce malformed HTML.

##### Send Attorney Review Email
- **Type & Technical Role:** `n8n-nodes-base.emailSend` (SMTP Integration)
- **Configuration:** Sends the review notification via SMTP credentials.
- **KeyExpressions:** `={{ $json.recipient }}`, `={{ $vars.FIRM_FROM_EMAIL }}`
- **Input / Output Connections:** Input: `Build Attorney Review Email` | Output: None (Block Terminal)
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** SMTP authentication failure or blocked relay ports.

---

### 2.3 Attorney Decision Processing

#### Overview
This block intercepts the attorney’s approval webhook request, validates link integrity, retrieves tracking entries from Google Sheets, transitions the draft DocuSign envelope to `sent` status, logs the update, and displays an HTML confirmation response.

#### Nodes Involved
- `Attorney Decision Request Trigger`
- `Parse Attorney Decision Request`
- `If Attorney Decision Valid`
- `Respond Invalid Attorney Decision`
- `Read Engagement Pipeline Log`
- `Find Engagement Row By Review Id`
- `If Engagement Row Found`
- `Respond Engagement Not Found`
- `Fetch DocuSign Account Info`
- `Transition Envelope Status To Sent`
- `Parse Envelope Send Response`
- `Update Engagement Row As Sent`
- `Respond Attorney Decision Confirmation`

#### Node Details

##### Attorney Decision Request Trigger
- **Type & Technical Role:** `n8n-nodes-base.webhook` (Webhook Trigger)
- **Configuration:** GET request on path `engagement-letter-decision` with `responseMode: responseNode`.
- **Key Expressions:** None
- **Input / Output Connections:** Input: None (Trigger) | Output: `Parse Attorney Decision Request`
- **Version-Specific Requirements:** v2.1
- **Edge Cases & Failure Types:** Missing query parameters.

##### Parse Attorney Decision Request
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Extracts `review_id` and `action` parameters from query strings, validating that action equals `approve`.
- **Key Expressions:** Checks `query.review_id` and `query.action`.
- **Input / Output Connections:** Input: `Attorney Decision Request Trigger` | Output: `If Attorney Decision Valid`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Missing query string elements.

##### If Attorney Decision Valid
- **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Router)
- **Configuration:** Assesses `is_valid` boolean flag.
- **Key Expressions:** `={{ $json.is_valid }}`
- **Input / Output Connections:** Input: `Parse Attorney Decision Request` | Output: `Read Engagement Pipeline Log` (True), `Respond Invalid Attorney Decision` (False)
- **Version-Specific Requirements:** v2.3
- **Edge Cases & Failure Types:** None.

##### Respond Invalid Attorney Decision
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook Response)
- **Configuration:** Returns an HTTP 400 error page informing the user that the link is invalid or expired.
- **Key Expressions:** Response code `400` with custom HTML payload.
- **Input / Output Connections:** Input: `If Attorney Decision Valid` (False) | Output: None (Terminal)
- **Version-Specific Requirements:** v1.1
- **Edge Cases & Failure Types:** None.

##### Read Engagement Pipeline Log
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Spreadsheet Integration)
- **Configuration:** Reads all rows from the "Engagement Pipeline" sheet.
- **Key Expressions:** `={{ $vars.ENGAGEMENT_PIPELINE_SHEET_ID }}`
- **Input / Output Connections:** Input: `If Attorney Decision Valid` (True) | Output: `Find Engagement Row By Review Id`
- **Version-Specific Requirements:** v4
- **Edge Cases & Failure Types:** API timeouts on large spreadsheets.

##### Find Engagement Row By Review Id
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Scans sheet rows to find a matching `review_id`.
- **Key Expressions:** Compares `row.review_id` against the target review ID.
- **Input / Output Connections:** Input: `Read Engagement Pipeline Log` | Output: `If Engagement Row Found`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Missing records resulting in unmatched states.

##### If Engagement Row Found
- **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Router)
- **Configuration:** Checks `is_found` boolean.
- **Key Expressions:** `={{ $json.is_found }}`
- **Input / Output Connections:** Input: `Find Engagement Row By Review Id` | Output: `Fetch DocuSign Account Info` (True), `Respond Engagement Not Found` (False)
- **Version-Specific Requirements:** v2.3
- **Edge Cases & Failure Types:** None.

##### Respond Engagement Not Found
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook Response)
- **Configuration:** Returns an HTTP 404 error page.
- **Key Expressions:** Response code `404` with notification text.
- **Input / Output Connections:** Input: `If Engagement Row Found` (False) | Output: None (Terminal)
- **Version-Specific Requirements:** v1.1
- **Edge Cases & Failure Types:** None.

##### Fetch DocuSign Account Info
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request)
- **Configuration:** GET request to OAuth userinfo endpoint to retrieve active account identifiers.
- **Key Expressions:** `={{ $vars.DOCUSIGN_AUTH_BASE_URL }}/oauth/userinfo`
- **Input / Output Connections:** Input: `If Engagement Row Found` (True) | Output: `Transition Envelope Status To Sent`
- **Version-Specific Requirements:** v4.4
- **Edge Cases & Failure Types:** Authentication failures.

##### Transition Envelope Status To Sent
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request)
- **Configuration:** PUT request to update envelope status to `sent`. Triggers DocuSign to email the client.
- **Key Expressions:** References envelope ID from prior search nodes.
- **Input / Output Connections:** Input: `Fetch DocuSign Account Info` | Output: `Parse Envelope Send Response`
- **Version-Specific Requirements:** v4.4
- **Edge Cases & Failure Types:** Envelope already sent or voided.

##### Parse Envelope Send Response
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Evaluates whether the send operation succeeded or returned an error code.
- **Key Expressions:** Checks for absence of `errorCode`.
- **Input / Output Connections:** Input: `Transition Envelope Status To Sent` | Output: `Update Engagement Row As Sent`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** API format variations.

##### Update Engagement Row As Sent
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Spreadsheet Integration)
- **Configuration:** Updates the matching row in Google Sheets with status (`sent` or `send_failed`) and timestamp using `review_id` as the matching key.
- **Key Expressions:** `={{ $now.toISO() }}`
- **Input / Output Connections:** Input: `Parse Envelope Send Response` | Output: `Respond Attorney Decision Confirmation`
- **Version-Specific Requirements:** v4
- **Edge Cases & Failure Types:** Sheet lock or update collision.

##### Respond Attorney Decision Confirmation
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook Response)
- **Configuration:** Returns an HTML confirmation page to the browser.
- **Key Expressions:** Dynamic inline HTML based on `sent_ok` status.
- **Input / Output Connections:** Input: `Update Engagement Row As Sent` | Output: None (Terminal)
- **Version-Specific Requirements:** v1.1
- **Edge Cases & Failure Types:** None.

---

### 2.4 DocuSign Signature Webhook Handling

#### Overview
This block acts as a webhook listener for DocuSign Connect events, filters out irrelevant status notifications, and isolates fully completed signing events.

#### Nodes Involved
- `When DocuSign Signature Received`
- `Parse DocuSign Signature Event`
- `If Envelope Fully Signed`
- `Skip Non Completion Event`
- `Respond Skipped Signature Event`

#### Node Details

##### When DocuSign Signature Received
- **Type & Technical Role:** `n8n-nodes-base.webhook` (Webhook Trigger)
- **Configuration:** HTTP POST endpoint on path `engagement-letter-signed` with `responseMode: responseNode`.
- **Key Expressions:** None
- **Input / Output Connections:** Input: None (Trigger) | Output: `Parse DocuSign Signature Event`
- **Version-Specific Requirements:** v2.1
- **Edge Cases & Failure Types:** Unverified payload signatures if DocuSign HMAC is enforced externally.

##### Parse DocuSign Signature Event
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Extracts `envelopeId` and event state from the payload. Sets `is_valid_signed` to true if the event matches completion patterns.
- **Key Expressions:** Regex check `/completed/i.test(event)`.
- **Input / Output Connections:** Input: `When DocuSign Signature Received` | Output: `If Envelope Fully Signed`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Malformed XML/JSON structures from non-standard Connect configurations.

##### If Envelope Fully Signed
- **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Router)
- **Configuration:** Evaluates `is_valid_signed`.
- **Key Expressions:** `={{ $json.is_valid_signed }}`
- **Input / Output Connections:** Input: `Parse DocuSign Signature Event` | Output: `Read Pipeline Log For Signature` (True), `Skip Non Completion Event` (False)
- **Version-Specific Requirements:** v2.3
- **Edge Cases & Failure Types:** None.

##### Skip Non Completion Event
- **Type & Technical Role:** `n8n-nodes-base.noOp` (Operation Placeholder)
- **Configuration:** Passthrough node for unhandled events.
- **Key Expressions:** None
- **Input / Output Connections:** Input: `If Envelope Fully Signed` (False) | Output: `Respond Skipped Signature Event`
- **Version-Specific Requirements:** v1
- **Edge Cases & Failure Types:** None.

##### Respond Skipped Signature Event
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook Response)
- **Configuration:** Returns an HTTP 200 "OK" response to DocuSign to prevent retries.
- **Key Expressions:** Response body `OK`.
- **Input / Output Connections:** Input: `Skip Non Completion Event` | Output: None (Terminal)
- **Version-Specific Requirements:** v1.1
- **Edge Cases & Failure Types:** None.

---

### 2.5 Signature Completion Processing & Matter Opening

#### Overview
This block matches signed envelopes back to the pipeline log, creates a new legal matter in Clio, updates the spreadsheet tracking record, notifies staff via email, and confirms webhook receipt.

#### Nodes Involved
- `Read Pipeline Log For Signature`
- `Find Signed Row By Envelope Id`
- `If Signed Row Found`
- `Respond Signed Row Not Found`
- `Create New Clio Matter`
- `Parse Clio Matter Creation Response`
- `Update Engagement Row As Signed`
- `Notify Staff Of Signed Engagement`
- `Respond Signature Processed`

#### Node Details

##### Read Pipeline Log For Signature
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Spreadsheet Integration)
- **Configuration:** Reads all rows from the "Engagement Pipeline" sheet.
- **Key Expressions:** `={{ $vars.ENGAGEMENT_PIPELINE_SHEET_ID }}`
- **Input / Output Connections:** Input: `If Envelope Fully Signed` (True) | Output: `Find Signed Row By Envelope Id`
- **Version-Specific Requirements:** v4
- **Edge Cases & Failure Types:** API limits.

##### Find Signed Row By Envelope Id
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Matches the signed `envelope_id` against stored records.
- **Key Expressions:** Compares `row.envelope_id`.
- **Input / Output Connections:** Input: `Read Pipeline Log For Signature` | Output: `If Signed Row Found`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** Envelope ID mismatch due to data truncation.

##### If Signed Row Found
- **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Router)
- **Configuration:** Assesses `is_found` boolean.
- **Key Expressions:** `={{ $json.is_found }}`
- **Input / Output Connections:** Input: `Find Signed Row By Envelope Id` | Output: `Create New Clio Matter` (True), `Respond Signed Row Not Found` (False)
- **Version-Specific Requirements:** v2.3
- **Edge Cases & Failure Types:** None.

##### Respond Signed Row Not Found
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook Response)
- **Configuration:** Returns an HTTP 200 "OK" response to DocuSign to prevent retry loops despite missing local context.
- **Key Expressions:** Response body `OK`.
- **Input / Output Connections:** Input: `If Signed Row Found` (False) | Output: None (Terminal)
- **Version-Specific Requirements:** v1.1
- **Edge Cases & Failure Types:** None.

##### Create New Clio Matter
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request)
- **Configuration:** POST request to `={{ $vars.CLIO_BASE_URL }}/api/v4/matters.json` using OAuth2. Creates an open matter assigned to the client contact ID and practice area.
- **Key Expressions:** JSON payload specifies client ID, practice area, and description.
- **Input / Output Connections:** Input: `If Signed Row Found` (True) | Output: `Parse Clio Matter Creation Response`
- **Version-Specific Requirements:** v4.4
- **Edge Cases & Failure Types:** Invalid contact ID reference in Clio.

##### Parse Clio Matter Creation Response
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
- **Configuration:** Validates matter creation success by checking for a matter ID in the response.
- **Key Expressions:** Checks `res.data.id`.
- **Input / Output Connections:** Input: `Create New Clio Matter` | Output: `Update Engagement Row As Signed`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** API error payloads.

##### Update Engagement Row As Signed
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Spreadsheet Integration)
- **Configuration:** Updates the matching row in Google Sheets with status (`signed_matter_opened` or `signed_matter_failed`), matter ID, and timestamp.
- **Key Expressions:** `={{ $now.toISO() }}`
- **Input / Output Connections:** Input: `Parse Clio Matter Creation Response` | Output: `Notify Staff Of Signed Engagement`
- **Version-Specific Requirements:** v4
- **Edge Cases & Failure Types:** Sheet write concurrency errors.

##### Notify Staff Of Signed Engagement
- **Type & Technical Role:** `n8n-nodes-base.emailSend` (SMTP Integration)
- **Configuration:** Sends an internal notification email to staff detailing the completed signing and newly opened Clio matter.
- **Key Expressions:** `={{ $vars.FIRM_ATTORNEY_EMAIL }}`
- **Input / Output Connections:** Input: `Update Engagement Row As Signed` | Output: `Respond Signature Processed`
- **Version-Specific Requirements:** v2
- **Edge Cases & Failure Types:** SMTP failures.

##### Respond Signature Processed
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook Response)
- **Configuration:** Returns an HTTP 200 "OK" response to DocuSign Connect.
- **Key Expressions:** Response body `OK`.
- **Input / Output Connections:** Input: `Notify Staff Of Signed Engagement` | Output: None (Terminal)
- **Version-SpecificRequirements:** v1.1
- **Edge Cases & Failure Types:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Setup Overview | None | None | ## Engagement Letter & E-Sign Pipeline... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Block 1 Documentation | None | None | ## Lead qualification trigger and contact check... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Block 2 Documentation | None | None | ## Engagement letter envelope creation... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Block 3 Documentation | None | None | ## Attorney decision processing... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Block 4 Documentation | None | None | ## DocuSign signature webhook handling... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Block 5 Documentation | None | None | ## Signature completion processing... |
| When Lead Qualified For Engagement | n8n-nodes-base.webhook | Ingests qualified lead payload | None | Parse Lead Qualification Data | ## Lead qualification trigger and contact check... |
| Parse Lead Qualification Data | n8n-nodes-base.code | Normalizes lead attributes | When Lead Qualified For Engagement | Fetch Existing Clio Contacts | ## Lead qualification trigger and contact check... |
| Fetch Existing Clio Contacts | n8n-nodes-base.httpRequest | Searches Clio contacts by email | Parse Lead Qualification Data | Determine If Clio Contact Exists | ## Lead qualification trigger and contact check... |
| Determine If Clio Contact Exists | n8n-nodes-base.code | Verifies exact email match | Fetch Existing Clio Contacts | If Clio Contact Exists | ## Lead qualification trigger and contact check... |
| If Clio Contact Exists | n8n-nodes-base.if | Evaluates contact existence | Determine If Clio Contact Exists | Use Existing Clio Contact Id, Create New Clio Contact | ## Lead qualification trigger and contact check... |
| Create New Clio Contact | n8n-nodes-base.httpRequest | Creates new Clio person contact | If Clio Contact Exists | Merge New Clio Contact Id | ## Lead qualification trigger and contact check... |
| Merge New Clio Contact Id | n8n-nodes-base.code | Merges new contact ID into context | Create New Clio Contact | Fetch DocuSign User Info | ## Lead qualification trigger and contact check... |
| Use Existing Clio Contact Id | n8n-nodes-base.code | Assigns existing contact ID | If Clio Contact Exists | Fetch DocuSign User Info | ## Lead qualification trigger and contact check... |
| Fetch DocuSign User Info | n8n-nodes-base.httpRequest | Resolves DocuSign account metadata | Merge New Clio Contact Id, Use Existing Clio Contact Id | Build DocuSign Envelope Request | ## Lead qualification trigger and contact check... |
| Build DocuSign Envelope Request | n8n-nodes-base.code | Formats draft envelope payload | Fetch DocuSign User Info | Post Draft Envelope To DocuSign | ## Engagement letter envelope creation... |
| Post Draft Envelope To DocuSign | n8n-nodes-base.httpRequest | Creates draft DocuSign envelope | Build DocuSign Envelope Request | Parse DocuSign Envelope Response | ## Engagement letter envelope creation... |
| Parse DocuSign Envelope Response | n8n-nodes-base.code | Validates envelope creation response | Post Draft Envelope To DocuSign | Assign Engagement Review Id | ## Engagement letter envelope creation... |
| Assign Engagement Review Id | n8n-nodes-base.code | Generates unique review tracking ID | Parse DocuSign Envelope Response | Append Engagement Review To Sheets | ## Engagement letter envelope creation... |
| Append Engagement Review To Sheets | n8n-nodes-base.googleSheets | Logs pending review in Google Sheets | Assign Engagement Review Id | Build Attorney Review Email | ## Engagement letter envelope creation... |
| Build Attorney Review Email | n8n-nodes-base.code | Constructs attorney review HTML email | Append Engagement Review To Sheets | Send Attorney Review Email | ## Engagement letter envelope creation... |
| Send Attorney Review Email | n8n-nodes-base.emailSend | Sends notification to attorney | Build Attorney Review Email | None | ## Engagement letter envelope creation... |
| Attorney Decision Request Trigger | n8n-nodes-base.webhook | Receives attorney decision webhook | None | Parse Attorney Decision Request | ## Attorney decision processing... |
| Parse Attorney Decision Request | n8n-nodes-base.code | Parses attorney decision parameters | Attorney Decision Request Trigger | If Attorney Decision Valid | ## Attorney decision processing... |
| If Attorney Decision Valid | n8n-nodes-base.if | Validates attorney action parameter | Parse Attorney Decision Request | Read Engagement Pipeline Log, Respond Invalid Attorney Decision | ## Attorney decision processing... |
| Respond Invalid Attorney Decision | n8n-nodes-base.respondToWebhook | Returns invalid link error response | If Attorney Decision Valid | None | ## Attorney decision processing... |
| Read Engagement Pipeline Log | n8n-nodes-base.googleSheets | Reads tracking sheet rows | If Attorney Decision Valid | Find Engagement Row By Review Id | ## Attorney decision processing... |
| Find Engagement Row By Review Id | n8n-nodes-base.code | Matches review ID against sheet log | Read Engagement Pipeline Log | If Engagement Row Found | ## Attorney decision processing... |
| If Engagement Row Found | n8n-nodes-base.if | Confirms engagement record existence | Find Engagement Row By Review Id | Fetch DocuSign Account Info, Respond Engagement Not Found | ## Attorney decision processing... |
| Respond Engagement Not Found | n8n-nodes-base.respondToWebhook | Returns not-found error response | If Engagement Row Found | None | ## Attorney decision processing... |
| Fetch DocuSign Account Info | n8n-nodes-base.httpRequest | Retrieves DocuSign account context | If Engagement Row Found | Transition Envelope Status To Sent | ## Attorney decision processing... |
| Transition Envelope Status To Sent | n8n-nodes-base.httpRequest | Updates envelope status to sent | Fetch DocuSign Account Info | Parse Envelope Send Response | ## Attorney decision processing... |
| Parse Envelope Send Response | n8n-nodes-base.code | Validates envelope send operation | Transition Envelope Status To Sent | Update Engagement Row As Sent | ## Attorney decision processing... |
| Update Engagement Row As Sent | n8n-nodes-base.googleSheets | Updates tracking log to sent status | Parse Envelope Send Response | Respond Attorney Decision Confirmation | ## Attorney decision processing... |
| Respond Attorney Decision Confirmation | n8n-nodes-base.respondToWebhook | Returns success/failure confirmation HTML | Update Engagement Row As Sent | None | ## Attorney decision processing... |
| When DocuSign Signature Received | n8n-nodes-base.webhook | Receives DocuSign Connect webhook | None | Parse DocuSign Signature Event | ## DocuSign signature webhook handling... |
| Parse DocuSign Signature Event | n8n-nodes-base.code | Parses and validates signature events | When DocuSign Signature Received | If Envelope Fully Signed | ## DocuSign signature webhook handling... |
| If Envelope Fully Signed | n8n-nodes-base.if | Filters completed signing events | Parse DocuSign Signature Event | Read Pipeline Log For Signature, Skip Non Completion Event | ## DocuSign signature webhook handling... |
| Skip Non Completion Event | n8n-nodes-base.noOp | Placeholder for ignored events | If Envelope Fully Signed | Respond Skipped Signature Event | ## DocuSign signature webhook handling... |
| Respond Skipped Signature Event | n8n-nodes-base.respondToWebhook | Acknowledges non-actionable webhook | Skip Non Completion Event | None | ## DocuSign signature webhook handling... |
| Read Pipeline Log For Signature | n8n-nodes-base.googleSheets | Reads tracking log for signature match | If Envelope Fully Signed | Find Signed Row By Envelope Id | ## Signature completion processing... |
| Find Signed Row By Envelope Id | n8n-nodes-base.code | Matches envelope ID in tracking log | Read Pipeline Log For Signature | If Signed Row Found | ## Signature completion processing... |
| If Signed Row Found | n8n-nodes-base.if | Confirms signed row match | Find Signed Row By Envelope Id | Create New Clio Matter, Respond Signed Row Not Found | ## Signature completion processing... |
| Respond Signed Row Not Found | n8n-nodes-base.respondToWebhook | Acknowledges webhook when row is missing | If Signed Row Found | None | ## Signature completion processing... |
| Create New Clio Matter | n8n-nodes-base.httpRequest | Opens new Clio matter | If Signed Row Found | Parse Clio Matter Creation Response | ## Signature completion processing... |
| Parse Clio Matter Creation Response | n8n-nodes-base.code | Parses Clio matter creation result | Create New Clio Matter | Update Engagement Row As Signed | ## Signature completion processing... |
| Update Engagement Row As Signed | n8n-nodes-base.googleSheets | Updates tracking log as signed & opened | Parse Clio Matter Creation Response | Notify Staff Of Signed Engagement | ## Signature completion processing... |
| Notify Staff Of Signed Engagement | n8n-nodes-base.emailSend | Sends staff notification email | Update Engagement Row As Signed | Respond Signature Processed | ## Signature completion processing... |
| Respond Signature Processed | n8n-nodes-base.respondToWebhook | Acknowledges DocuSign webhook | Notify Staff Of Signed Engagement | None | ## Signature completion processing... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Environment Variables & Credentials
1. Configure **Clio API (OAuth2)** credentials and record your credential ID.
2. Configure **DocuSign account (OAuth2)** credentials and record your credential ID.
3. Configure **Email account (SMTP)** credentials.
4. Set up the following workflow variables in your n8n instance:
   - `CLIO_BASE_URL`: Base URL for your Clio region (e.g., `https://app.clio.com`)
   - `DOCUSIGN_AUTH_BASE_URL`: DocuSign authentication URL (e.g., `https://account.docusign.com` or sandbox equivalent)
   - `ENGAGEMENT_LETTER_TEMPLATE_ID`: The DocuSign Template ID for engagement letters
   - `ENGAGEMENT_LETTER_ROLE_NAME`: The assigned signer role name in your DocuSign template
   - `ENGAGEMENT_PIPELINE_SHEET_ID`: Google Sheet ID containing an "Engagement Pipeline" tab
   - `FIRM_FROM_EMAIL`: Sender address for outgoing internal emails
   - `FIRM_ATTORNEY_EMAIL`: Default notification recipient email
   - `ENGAGEMENT_DECISION_WEBHOOK_URL`: Production webhook URL for attorney decisions (configured after activation)

---

#### Step 2: Build Block 1 (Input Reception & Contact Management)
1. **Webhook Node (`When Lead Qualified For Engagement`)**:
   - Set Type to Webhook, Method: `POST`, Path: `engagement-letter-intake`.
2. **Code Node (`Parse Lead Qualification Data`)**:
   - Extract lead attributes (`full_name`, `email`, `phone`, `practice_area`, `case_details`, `attorney_email`).
3. **HTTP Request Node (`Fetch Existing Clio Contacts`)**:
   - Method: `GET`, URL: `={{ $vars.CLIO_BASE_URL }}/api/v4/contacts.json`.
   - Add query parameter `query` set to `={{ $json.email }}`.
   - Configure Generic OAuth2 authentication using your Clio credentials.
4. **Code Node (`Determine If Clio Contact Exists`)**:
   - Match results against email to locate existing contact IDs.
5. **If Node (`If Clio Contact Exists`)**:
   - Evaluate `={{ !!$json.existing_contact_id }}`.
6. **HTTP Request Node (`Create New Clio Contact`)**:
   - Connect to False branch of If node. Method: `POST`, URL: `={{ $vars.CLIO_BASE_URL }}/api/v4/contacts.json`.
   - Send JSON body specifying `type: 'Person'`, name, email, and phone.
7. **Code Node (`Merge New Clio Contact Id`)**:
   - Combine new contact ID with lead context.
8. **Code Node (`Use Existing Clio Contact Id`)**:
   - Connect to True branch of If node; map existing ID.
9. **HTTP Request Node (`DocuSign User Info`)**:
   - Connect both contact resolution branches to this node. Method: `GET`, URL: `={{ $vars.DOCUSIGN_AUTH_BASE_URL }}/oauth/userinfo`. Use DocuSign OAuth2 credentials.

---

#### Step 3: Build Block 2 (Engagement Letter Envelope Creation)
1. **Code Node (`Build DocuSign Envelope Request`)**:
   - Format template roles and text tabs (`full_name`, `email`, `phone`, `practice_area`, `case_details`, `date`). Set status to `created`.
2. **HTTP Request Node (`Post Draft Envelope To DocuSign`)**:
   - Method: `POST`, URL: `={{ $json.docusign_base_uri }}/restapi/v2.1/accounts/{{ $json.docusign_account_id }}/envelopes`. Send JSON body.
3. **Code Node (`Parse DocuSign Envelope Response`)**:
   - Validate presence of `envelopeId`.
4. **Code Node (`Assign Engagement Review Id`)**:
   - Generate unique review ID (`ENGL-<timestamp>`).
5. **Google Sheets Node (`Append Engagement Review To Sheets`)**:
   - Operation: `append`, Document ID: `={{ $vars.ENGAGEMENT_PIPELINE_SHEET_ID }}`, Sheet Name: `Engagement Pipeline`. Map columns (`review_id`, `envelope_id`, `status`, `full_name`, `email`, `practice_area`, `contact_id`, `created_at`).
6. **Code Node (`Build Attorney Review Email`)**:
   - Generate HTML approval email incorporating the review ID decision link.
7. **Email Send Node (`Send Attorney Review Email`)**:
   - Use SMTP credentials to send the review email to the attorney.

---

#### Step 4: Build Block 3 (Attorney Decision Processing)
1. **Webhook Node (`Attorney Decision Request Trigger`)**:
   - Method: `GET`, Path: `engagement-letter-decision`, Response Mode: `Response Node`.
2. **Code Node (`Parse Attorney Decision Request`)**:
   - Extract `review_id` and `action` from query parameters.
3. **If Node (`If Attorney Decision Valid`)**:
   - Evaluate `={{ $json.is_valid }}`.
4. **Respond to Webhook Node (`Respond Invalid Attorney Decision`)**:
   - Connect to False branch. Response Code: `400`, Body: invalid link message.
5. **Google Sheets Node (`Read Engagement Pipeline Log`)**:
   - Connect to True branch. Operation: `read`, Sheet: `Engagement Pipeline`.
6. **Code Node (`Find Engagement Row By Review Id`)**:
   - Match row by `review_id`.
7. **If Node (`If Engagement Row Found`)**:
   - Evaluate `={{ $json.is_found }}`.
8. **Respond to Webhook Node (`Respond Engagement Not Found`)**:
   - Connect to False branch. Response Code: `404`, Body: not found message.
9. **HTTP Request Node (`Fetch DocuSign Account Info`)**:
   - Connect to True branch. GET request to OAuth userinfo endpoint.
10. **HTTP Request Node (`Transition Envelope Status To Sent`)**:
    - Method: `PUT`, URL pointing to the envelope endpoint, sending JSON body `{ status: 'sent' }`.
11. **Code Node (`Parse Envelope Send Response`)**:
    - Validate send operation success.
12. **Google Sheets Node (`Update Engagement Row As Sent`)**:
    - Operation: `appendOrUpdate`, matching on `review_id`. Update `status` and `sent_at`.
13. **Respond to Webhook Node (`Respond Attorney Decision Confirmation`)**:
    - Return confirmation HTML to browser.

---

#### Step 5: Build Block 4 (DocuSign Signature Webhook Handling)
1. **Webhook Node (`When DocuSign Signature Received`)**:
   - Method: `POST`, Path: `engagement-letter-signed`, Response Mode: `Response Node`.
2. **Code Node (`Parse DocuSign Signature Event`)**:
   - Extract envelope ID and check completion event regex.
3. **If Node (`If Envelope Fully Signed`)**:
   - Evaluate `={{ $json.is_valid_signed }}`.
4. **No-Op Node (`Skip Non Completion Event`)**:
   - Connect to False branch.
5. **Respond to Webhook Node (`Respond Skipped Signature Event`)**:
   - Respond with text `OK`.

---

#### Step 6: Build Block 5 (Signature Completion Processing & Matter Opening)
1. **Google Sheets Node (`Read Pipeline Log For Signature`)**:
   - Connect to True branch of signature If node. Operation: `read`, Sheet: `Engagement Pipeline`.
2. **Code Node (`Find Signed Row By Envelope Id`)**:
   - Match signed envelope ID against sheet rows.
3. **If Node (`If Signed Row Found`)**:
   - Evaluate `={{ $json.is_found }}`.
4. **Respond to Webhook Node (`Respond Signed Row Not Found`)**:
   - Connect to False branch. Respond with text `OK`.
5. **HTTP Request Node (`Create New Clio Matter`)**:
   - Connect to True branch. Method: `POST`, URL: `={{ $vars.CLIO_BASE_URL }}/api/v4/matters.json`. Send JSON payload with client ID, practice area, and status `Open`.
6. **Code Node (`Parse Clio Matter Creation Response`)**:
   - Validate matter creation ID.
7. **Google Sheets Node (`Update Engagement Row As Signed`)**:
   - Operation: `appendOrUpdate`, matching on `review_id`. Update `status`, `matter_id`, and `signed_at`.
8. **Email Send Node (`Notify Staff Of Signed Engagement`)**:
   - Send internal staff notification via SMTP.
9. **Respond to Webhook Node (`Respond Signature Processed`)**:
   - Respond with text `OK` to DocuSign Connect.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Clio API Integration Guidelines | Consult the [Clio API v4 Documentation](https://app.clio.com/api/v4/documentation) for schema definitions and rate limits. |
| DocuSign REST API Reference | Refer to the [DocuSign eSignature REST API Guide](https://developers.docusign.com/docs/esign-rest-api/) for envelope workflows and template configuration. |
| Google Sheets Integration Setup | Ensure service accounts or OAuth connections have read/write permissions to the target spreadsheet and that column headers match required keys (`review_id`, `envelope_id`, `status`, `sent_at`, `signed_at`, `contact_id`, `matter_id`). |