Send Jira support auto-replies and CSAT surveys via WhatsApp, Gmail, and HubSpot

https://n8nworkflows.xyz/workflows/send-jira-support-auto-replies-and-csat-surveys-via-whatsapp--gmail--and-hubspot-19609


# Send Jira support auto-replies and CSAT surveys via WhatsApp, Gmail, and HubSpot

### 1. Workflow Overview

This workflow automates the customer support lifecycle for Jira tickets across three distinct functional paths:
1. **Auto-Reply Path:** Triggers when a parent workflow invokes it, evaluates customer contact channels (WhatsApp or Gmail), sends an automatic acknowledgement message, logs a comment and label in Jira, and optionally creates a synced note in HubSpot.
2. **CSAT Request Path:** Listens for Jira webhooks when a ticket transitions to a "Done" status, verifies webhook authentication and ticket criteria, dynamically generates a CSAT survey link, delivers the survey via WhatsApp or Gmail, and records the sent status back to Jira.
3. **CSAT Submission Path:** Captures form submissions from customers, validates the rating and ticket keys against Jira, writes the CSAT score and feedback as a comment (and updates Jira custom fields), logs the result in HubSpot, and renders a completion page.

#### Functional Blocks:
- **1.1 Auto-Reply Entry & Normalization:** Receives sub-workflow payloads, defines core configurations, and standardizes customer contact details.
- **1.2 Auto-Reply Delivery & Logging:** Routes messages via WhatsApp (WAHA) or Gmail, updates Jira issues with comments/labels, and syncs interactions to HubSpot CRM.
- **1.3 CSAT Webhook Intake & Validation:** Receives Jira issue status updates via webhook, validates webhook security tokens, and sets form paths.
- **1.4 CSAT Survey Dispatch:** Fetches resolved issue details, evaluates dispatch criteria, and delivers CSAT survey invites through WhatsApp or Gmail, updating Jira tracking labels.
- **1.5 CSAT Form Processing & CRM Sync:** Captures user feedback from the n8n form interface, updates Jira issue metrics, and creates/associates corresponding HubSpot CRM notes.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Auto-Reply Entry & Normalization
**Overview:** This block initializes the auto-reply lifecycle by accepting execution triggers from parent workflows, loading centralized configuration parameters, and normalizing inbound customer data.

**Nodes Involved:**
- `Trigger Workflow Execution`
- `Load Auto Reply Config`
- `Normalize Auto Reply Payload`
- `Evaluate Auto Reply Conditions`

**Node Details:**
- **Trigger Workflow Execution**
  - *Type & Technical Role:* `n8n-nodes-base.executeWorkflowTrigger` (Sub-workflow entry point)
  - *Configuration:* Input source set to passthrough.
  - *Connections:* Input: None (Trigger); Output: `Load Auto Reply Config`.
  - *Edge Cases:* Missing payload attributes from parent workflows will result in downstream validation catches.

- **Load Auto Reply Config**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation and configuration loader)
  - *Configuration:* Injects a centralized JSON constant (`CLOSE_LOOP`) defining project keys, status names, webhook secrets, API base URLs, custom field mappings, and Jira label names into the execution flow.
  - *Connections:* Input: `Trigger Workflow Execution`; Output: `Normalize Auto Reply Payload`.

- **Normalize Auto Reply Payload**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data normalization and validation prep)
  - *Configuration:* Sanitizes incoming parameters, formats phone numbers into valid WhatsApp chat IDs (`@c.us`), resolves WAHA session identifiers, and constructs the outbound auto-reply text template.
  - *Key Expressions:* Accesses configuration flags and validates `jiraKey`, `email`, and `whatsappChatId`.
  - *Connections:* Input: `Load Auto Reply Config`; Output: `Evaluate Auto Reply Conditions`.

- **Evaluate Auto Reply Conditions**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional routing)
  - *Configuration:* Evaluates the boolean expression `{{ $json.canSend }}` to determine if valid contact criteria (email or WhatsApp phone/ID) are present.
  - *Connections:* Input: `Normalize Auto Reply Payload`; Output (True): `Check WhatsApp Channel`; Output (False): Bypassed/terminated.
  - *Edge Cases:* Unrecognized channels or missing recipient addresses route execution down failure handling paths.

---

#### Block 1.2: Auto-Reply Delivery & Logging
**Overview:** Handles the dispatch of customer auto-replies over WhatsApp or Gmail, logs audit trails directly to Jira via comments and labels, and synchronizes interaction notes with HubSpot CRM.

**Nodes Involved:**
- `Check WhatsApp Channel`
- `Send Auto Reply via WhatsApp`
- `Email Auto Reply via Gmail`
- `Check HubSpot Note Eligibility`
- `Build Note Data for HubSpot`
- `Create Note in HubSpot`
- `Prepare HubSpot Association`
- `Associate Note in HubSpot`
- `Prepare Auto Reply Mark`
- `Comment Jira Auto Reply`
- `Label Jira Auto Reply Sent`

**Node Details:**
- **Check WhatsApp Channel**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Channel router)
  - *Configuration:* Evaluates whether `{{ $json.channel }}` equals `"whatsapp"`.
  - *Connections:* Input: `Evaluate Auto Reply Conditions`; Output 0 (True): `Send Auto Reply via WhatsApp`; Output 1 (False): `Email Auto Reply via Gmail`.

- **Send Auto Reply via WhatsApp**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
  - *Configuration:* Sends a POST request to the WAHA API endpoint (`/api/sendText`) using HTTP Header Authentication.
  - *Key Expressions:* Dynamically sets session ID, recipient chat ID, and message text from upstream JSON parameters.
  - *Connections:* Input: `Check WhatsApp Channel`; Output: `Check HubSpot Note Eligibility`.

- **Email Auto Reply via Gmail**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email integration)
  - *Configuration:* Sends plain-text email notifications using Gmail OAuth2 credentials.
  - *Key Expressions:* Recipient set to `{{ $json.email }}`; Subject uses `{{ $json.emailSubject }}`; Body uses `{{ $json.messageText }}`.
  - *Connections:* Input: `Check WhatsApp Channel`; Output: `Check HubSpot Note Eligibility`.

- **Check HubSpot Note Eligibility**
  - *Type & Technical Role:* `n8n-nodes-base.if` (CRM sync gating)
  - *Configuration:* Verifies if `{{ $json.hubspotContactId }}` is present and non-empty.
  - *Connections:* Input: `Send Auto Reply via WhatsApp` or `Email Auto Reply via Gmail`; Output (True): `Build Note Data for HubSpot`; Output (False): `Prepare Auto Reply Mark`.

- **Build Note Data for HubSpot**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Payload builder)
  - *Configuration:* Compiles support ticket metadata into a formatted string payload suitable for HubSpot note properties.
  - *Connections:* Input: `Check HubSpot Note Eligibility`; Output: `Create Note in HubSpot`.

- **Create Note in HubSpot**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (CRM API integration)
  - *Configuration:* POST request to HubSpot CRM v3 API (`/crm/v3/objects/notes`) using HTTP Header Auth.
  - *Connections:* Input: `Build Note Data for HubSpot`; Output: `Prepare HubSpot Association`.

- **Prepare HubSpot Association**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data mapping)
  - *Configuration:* Formats association payloads linking newly created CRM notes to contact records using v4 association schema.
  - *Connections:* Input: `Create Note in HubSpot`; Output: `Associate Note in HubSpot`.

- **Associate Note in HubSpot**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (CRM API integration)
  - *Configuration:* POST request to HubSpot v4 batch association API (`/crm/v4/associations/notes/contacts/batch/create`).
  - *Connections:* Input: `Prepare HubSpot Association`; Output: `Prepare Auto Reply Mark`.

- **Prepare Auto Reply Mark**
  - *Type & Technical Role:* Atlassian Jira integration preparation code node.
  - *Configuration:* Generates Atlassian Document Format (ADF) comment bodies and label update payloads.
  - *Connections:* Input: `Check HubSpot Note Eligibility` or `Associate Note in HubSpot`; Output: `Comment Jira Auto Reply`.

- **Comment Jira Auto Reply**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Jira integration)
  - *Configuration:* POST request to Jira REST API (`/rest/api/3/issue/{key}/comment`) using HTTP Basic Authentication.
  - *Connections:* Input: `Prepare Auto Reply Mark`; Output: `Label Jira Auto Reply Sent`.

- **Label Jira Auto Reply Sent**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Jira integration)
  - *Configuration:* PUT request to Jira REST API (`/rest/api/3/issue/{key}`) applying the `auto-reply-sent` label.
  - *Connections:* Input: `Comment Jira Auto Reply`; Output: None (Terminal node for this path).

---

#### Block 1.3: CSAT Webhook Intake & Validation
**Overview:** Receives webhook payloads from Jira when issue statuses change, validates security tokens, and establishes form routing paths.

**Nodes Involved:**
- `CSAT Webhook Listener`
- `Verify Webhook Token`
- `Configure CSAT Form Path`
- `Parse CSAT Webhook Data`
- `Verify CSAT Webhook`

**Node Details:**
- **CSAT Webhook Listener**
  - *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook trigger)
  - *Configuration:* Listens for POST requests at path `jira-csat`.
  - *Connections:* Input: None (Trigger); Output: `Verify Webhook Token`.

- **Verify Webhook Token**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Security verification)
  - *Configuration:* Validates that incoming query parameters (`token`) or headers (`x-webhook-token`) match the configured `webhookSecret`.
  - *Connections:* Input: `CSAT Webhook Listener`; Output (True): `Configure CSAT Form Path`; Output (False): Rejected/terminated.

- **Configure CSAT Form Path**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Configuration utility)
  - *Configuration:* Appends the active n8n form webhook path identifier (`CSAT_FORM_WEBHOOK_PATH`) to incoming execution payloads.
  - *Connections:* Input: `Verify Webhook Token`; Output: `Parse CSAT Webhook Data`.
  - *Edge Cases:* Requires manual updates to the code string if the form trigger UUID changes upon workflow re-import.

- **Parse CSAT Webhook Data**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Payload parser)
  - *Configuration:* Extracts Jira issue keys, status changes from changelogs, and checks whether issues have transitioned to the target "Done" state without prior CSAT tagging.
  - *Connections:* Input: `Configure CSAT Form Path`; Output: `Verify CSAT Webhook`.

- **Verify CSAT Webhook**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Execution gatekeeper)
  - *Configuration:* Evaluates `{{ $json.canProceed }}` to ensure valid state transitions before dispatching external HTTP calls.
  - *Connections:* Input: `Parse CSAT Webhook Data`; Output (True): `Build CSAT Fetch Query`; Output (False): Terminated.

---

#### Block 1.4: CSAT Survey Dispatch
**Overview:** Retrieves complete Jira issue properties for resolved tickets, validates dispatch criteria, and delivers CSAT survey invitations via WhatsApp or Gmail while updating Jira issue metadata.

**Nodes Involved:**
- `Build CSAT Fetch Query`
- `Fetch CSAT Done Issue from Jira`
- `Enrich CSAT Data`
- `Evaluate CSAT Sending Criteria`
- `Determine CSAT Channel`
- `Send CSAT via WhatsApp`
- `Email CSAT via Gmail`
- `Prepare CSAT Sent Mark`
- `Comment Jira CSAT Sent`
- `Label Jira Issue CSAT Sent`

**Node Details:**
- **Build CSAT Fetch Query**
  - *Type & Technical Role:* `n8n-nodes-base.code` (URL builder)
  - *Configuration:* Constructs the Jira REST API URL (`/rest/api/3/issue/{key}?fields=...`) incorporating custom fields for contact channels and tracking properties.
  - *Connections:* Input: `Verify CSAT Webhook`; Output: `Fetch CSAT Done Issue from Jira`.

- **Fetch CSAT Done Issue from Jira**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
  - *Configuration:* GET request to Jira REST API using HTTP Basic Auth with automatic retry and error pass-through.
  - *Connections:* Input: `Build CSAT Fetch Query`; Output: `Enrich CSAT Data`.

- **Enrich CSAT Data**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data enricher)
  - *Configuration:* Extracts channel types, phone numbers, customer emails, and HubSpot contact IDs from Jira custom fields; generates personalized CSAT survey URLs containing the n8n form path and ticket reference.
  - *Connections:* Input: `Fetch CSAT Done Issue from Jira`; Output: `Evaluate CSAT Sending Criteria`.

- **Evaluate CSAT Sending Criteria**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Validation gate)
  - *Configuration:* Checks `{{ $json.canSend }}` to ensure required channel identifiers are populated.
  - *Connections:* Input: `Enrich CSAT Data`; Output (True): `Determine CSAT Channel`; Output (False): Terminated.

- **Determine CSAT Channel**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Channel routing)
  - *Configuration:* Evaluates if `{{ $json.channel }}` equals `"whatsapp"`.
  - *Connections:* Input: `Evaluate CSAT Sending Criteria`; Output 0 (True): `Send CSAT via WhatsApp`; Output 1 (False): `Email CSAT via Gmail`.

- **Send CSAT via WhatsApp**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
  - *Configuration:* POST request to WAHA endpoint (`/api/sendText`) using HTTP Header Auth.
  - *Connections:* Input: `Determine CSAT Channel`; Output: `Prepare CSAT Sent Mark`.

- **Email CSAT via Gmail**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email integration)
  - *Configuration:* Sends survey invitation emails via Gmail OAuth2.
  - *Connections:* Input: `Determine CSAT Channel`; Output: `Prepare CSAT Sent Mark`.

- **Prepare CSAT Sent Mark**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Payload builder)
  - *Configuration:* Prepares ADF comment bodies indicating survey dispatch times and form URLs, alongside label update payload structures (`csat-sent`).
  - *Connections:* Input: `Send CSAT via WhatsApp` or `Email CSAT via Gmail`; Output: `Comment Jira CSAT Sent`.

- **Comment Jira CSAT Sent**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Jira integration)
  - *Configuration:* POST request to Jira REST API (`/rest/api/3/issue/{key}/comment`) via HTTP Basic Auth.
  - *Connections:* Input: `Prepare CSAT Sent Mark`; Output: `Label Jira Issue CSAT Sent`.

- **Label Jira Issue CSAT Sent**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Jira integration)
  - *Configuration:* PUT request to Jira REST API updating labels to include `csat-sent`.
  - *Connections:* Input: `Comment Jira CSAT Sent`; Output: None (Terminal node for this path).

---

#### Block 1.5: CSAT Form Processing & CRM Sync
**Overview:** Captures and validates user feedback submitted via the n8n form interface, updates Jira issue metrics and comments, synchronizes submission notes to HubSpot CRM, and returns confirmation messages.

**Nodes Involved:**
- `CSAT Form Submission Trigger`
- `Normalize CSAT Form Data`
- `Retrieve Jira CSAT Issue`
- `Fetch CSAT Issue from Jira`
- `Extract CSAT Details from Jira`
- `Prepare CSAT for Saving`
- `Post CSAT Score Comment`
- `Verify CSAT Custom Field`
- `Update Jira CSAT Score`
- `Check HubSpot CSAT Eligibility`
- `Prepare HubSpot Note Data`
- `Create CSAT Note on HubSpot`
- `Prepare HubSpot Association Data`
- `Post CSAT Note Association`
- `Capture CSAT Form Response`

**Node Details:**
- **CSAT Form Submission Trigger**
  - *Type & Technical Role:* `n8n-nodes-base.formTrigger` (Form webhook trigger)
  - *Configuration:* Hosts an interactive web form accepting rating scores (1–5), optional text comments, and pre-filled ticket query parameters.
  - *Connections:* Input: None (Trigger); Output: `Normalize CSAT Form Data`.

- **Normalize CSAT Form Data**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Validation utility)
  - *Configuration:* Validates that incoming ticket keys match pattern requirements (`KAN-\d+`) and verifies that scores fall between 1 and 5. Throws errors on invalid inputs.
  - *Connections:* Input: `CSAT Form Submission Trigger`; Output: `Retrieve Jira CSAT Issue`.

- **Retrieve Jira CSAT Issue**
  - *Type & Technical Role:* `n8n-nodes-base.code` (URL builder)
  - *Configuration:* Constructs Jira issue retrieval URLs requesting summary, labels, and relevant custom fields.
  - *Connections:* Input: `Normalize CSAT Form Data`; Output: `Fetch CSAT Issue from Jira`.

- **Fetch CSAT Issue from Jira**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
  - *Configuration:* GET request to Jira REST API via HTTP Basic Auth.
  - *Connections:* Input: `Retrieve Jira CSAT Issue`; Output: `Extract CSAT Details from Jira`.

- **Extract CSAT Details from Jira**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data extractor)
  - *Configuration:* Parses HubSpot contact associations from Jira fields.
  - *Connections:* Input: `Fetch CSAT Issue from Jira`; Output: `Prepare CSAT for Saving`.

- **Prepare CSAT for Saving**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Payload builder)
  - *Configuration:* Constructs Atlassian Document Format (ADF) comment bodies containing CSAT scores and feedback, and builds custom field update payloads.
  - *Connections:* Input: `Extract CSAT Details from Jira`; Output: `Post CSAT Score Comment`.

- **Post CSAT Score Comment**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Jira integration)
  - *Configuration:* POST request to Jira REST API (`/rest/api/3/issue/{key}/comment`) using HTTP Basic Auth.
  - *Connections:* Input: `Prepare CSAT for Saving`; Output: `Verify CSAT Custom Field`.

- **Verify CSAT Custom Field**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional router)
  - *Configuration:* Evaluates whether custom field update payloads (`_jiraFieldsBody`) are configured and present.
  - *Connections:* Input: `Post CSAT Score Comment`; Output 0 (True): `Update Jira CSAT Score`; Output 1 (False): `Check HubSpot CSAT Eligibility`.

- **Update Jira CSAT Score**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Jira integration)
  - *Configuration:* PUT request to Jira REST API updating designated custom field values with numeric scores.
  - *Connections:* Input: `Verify CSAT Custom Field`; Output: `Check HubSpot CSAT Eligibility`.

- **Check HubSpot CSAT Eligibility**
  - *Type & Technical Role:* `n8n-nodes-base.if` (CRM gating)
  - *Configuration:* Verifies if `{{ $json.hubspotContactId }}` exists.
  - *Connections:* Input: `Update Jira CSAT Score` or `Verify CSAT Custom Field`; Output (True): `Prepare HubSpot Note Data`; Output (False): `Capture CSAT Form Response`.

- **Prepare HubSpot Note Data**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Payload builder)
  - *Configuration:* Formats customer satisfaction scores and feedback comments into HubSpot note property bodies.
  - *Connections:* Input: `Check HubSpot CSAT Eligibility`; Output: `Create CSAT Note on HubSpot`.

- **Create CSAT Note on HubSpot**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (CRM API integration)
  - *Configuration:* POST request to HubSpot notes endpoint (`/crm/v3/objects/notes`) using HTTP Header Auth.
  - *Connections:* Input: `Prepare HubSpot Note Data`; Output: `Prepare HubSpot Association Data`.

- **Prepare HubSpot Association Data**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data mapper)
  - *Configuration:* Formats relationship associations linking created notes to HubSpot contact records.
  - *Connections:* Input: `Create CSAT Note on HubSpot`; Output: `Post CSAT Note Association`.

- **Post CSAT Note Association**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (CRM API integration)
  - *Configuration:* POST request to HubSpot v4 batch association API.
  - *Connections:* Input: `Prepare HubSpot Association Data`; Output: `Capture CSAT Form Response`.

- **Capture CSAT Form Response**
  - *Type & Technical Role:* `n8n-nodes-base.form` (Form completion handler)
  - *Configuration:* Renders an HTML completion screen ("Thank you") confirming submission success.
  - *Connections:* Input: `Check HubSpot CSAT Eligibility` or `Post CSAT Note Association`; Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## AI Customer Support – Customer Close-Loop... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Close-loop entry config... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Normalize auto-reply payload... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Select reply channel... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Prepare HubSpot logging... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Create reply note... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Mark auto-reply sent... |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## CSAT webhook intake... |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Parse CSAT webhook... |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Fetch completed issue... |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Deliver CSAT invite... |
| Sticky Note11 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Mark CSAT sent... |
| Sticky Note12 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## CSAT form intake... |
| Sticky Note13 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Load submitted issue... |
| Sticky Note14 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Save CSAT in Jira... |
| Sticky Note15 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Create CSAT note... |
| Sticky Note16 | n8n-nodes-base.stickyNote | Documentation & Setup Guide | — | — | ## Associate CSAT note... |
| Configure CSAT Form Path | n8n-nodes-base.code | Configuration injection | Verify Webhook Token | Parse CSAT Webhook Data | ## CSAT webhook intake... |
| Verify CSAT Webhook | n8n-nodes-base.if | Conditional verification | Parse CSAT Webhook Data | Build CSAT Fetch Query | ## Parse CSAT webhook... |
| Enrich CSAT Data | n8n-nodes-base.code | Data enrichment | Fetch CSAT Done Issue from Jira | Evaluate CSAT Sending Criteria | ## Fetch completed issue... |
| Fetch CSAT Done Issue from Jira | n8n-nodes-base.httpRequest | Jira REST API client | Build CSAT Fetch Query | Enrich CSAT Data | ## Fetch completed issue... |
| Build CSAT Fetch Query | n8n-nodes-base.code | URL builder | Verify CSAT Webhook | Fetch CSAT Done Issue from Jira | ## Parse CSAT webhook... |
| Capture CSAT Form Response | n8n-nodes-base.form | Form completion screen | Post CSAT Note Association, Check HubSpot CSAT Eligibility | — | ## Associate CSAT note... |
| Post CSAT Note Association | n8n-nodes-base.httpRequest | CRM API client | Prepare HubSpot Association Data | Capture CSAT Form Response | ## Associate CSAT note... |
| Prepare HubSpot Association Data | n8n-nodes-base.code | Payload mapping | Create CSAT Note on HubSpot | Post CSAT Note Association | ## Create CSAT note... |
| Create CSAT Note on HubSpot | n8n-nodes-base.httpRequest | CRM API client | Prepare HubSpot Note Data | Prepare HubSpot Association Data | ## Create CSAT note... |
| Prepare HubSpot Note Data | n8n-nodes-base.code | Payload builder | Check HubSpot CSAT Eligibility | Create CSAT Note on HubSpot | ## Save CSAT in Jira... |
| Check HubSpot CSAT Eligibility | n8n-nodes-base.if | CRM gating router | Update Jira CSAT Score, Verify CSAT Custom Field | Prepare HubSpot Note Data, Capture CSAT Form Response | ## Save CSAT in Jira... |
| Update Jira CSAT Score | n8n-nodes-base.httpRequest | Jira REST API client | Verify CSAT Custom Field | Check HubSpot CSAT Eligibility | ## Save CSAT in Jira... |
| Verify CSAT Custom Field | n8n-nodes-base.if | Conditional routing | Post CSAT Score Comment | Update Jira CSAT Score, Check HubSpot CSAT Eligibility | ## Save CSAT in Jira... |
| Post CSAT Score Comment | n8n-nodes-base.httpRequest | Jira REST API client | Prepare CSAT for Saving | Verify CSAT Custom Field | ## Save CSAT in Jira... |
| Prepare CSAT for Saving | n8n-nodes-base.code | Payload builder | Extract CSAT Details from Jira | Post CSAT Score Comment | ## Save CSAT in Jira... |
| Extract CSAT Details from Jira | n8n-nodes-base.code | Data extractor | Fetch CSAT Issue from Jira | Prepare CSAT for Saving | ## Load submitted issue... |
| Fetch CSAT Issue from Jira | n8n-nodes-base.httpRequest | Jira REST API client | Retrieve Jira CSAT Issue | Extract CSAT Details from Jira | ## Load submitted issue... |
| Retrieve Jira CSAT Issue | n8n-nodes-base.code | URL builder | Normalize CSAT Form Data | Fetch CSAT Issue from Jira | ## Load submitted issue... |
| Normalize CSAT Form Data | n8n-nodes-base.code | Form validation | CSAT Form Submission Trigger | Retrieve Jira CSAT Issue | ## CSAT form intake... |
| CSAT Form Submission Trigger | n8n-nodes-base.formTrigger | Webform entry point | — | Normalize CSAT Form Data | ## CSAT form intake... |
| Label Jira Issue CSAT Sent | n8n-nodes-base.httpRequest | Jira REST API client | Comment Jira CSAT Sent | — | ## Mark CSAT sent... |
| Comment Jira CSAT Sent | n8n-nodes-base.httpRequest | Jira REST API client | Prepare CSAT Sent Mark | Label Jira Issue CSAT Sent | ## Mark CSAT sent... |
| Prepare CSAT Sent Mark | n8n-nodes-base.code | Payload builder | Send CSAT via WhatsApp, Email CSAT via Gmail | Comment Jira CSAT Sent | ## Mark CSAT sent... |
| Send CSAT via WhatsApp | n8n-nodes-base.httpRequest | WAHA API client | Determine CSAT Channel | Prepare CSAT Sent Mark | ## Deliver CSAT invite... |
| Email CSAT via Gmail | n8n-nodes-base.gmail | Gmail integration | Determine CSAT Channel | Prepare CSAT Sent Mark | ## Deliver CSAT invite... |
| Determine CSAT Channel | n8n-nodes-base.if | Channel router | Evaluate CSAT Sending Criteria | Send CSAT via WhatsApp, Email CSAT via Gmail | ## Deliver CSAT invite... |
| Evaluate CSAT Sending Criteria | n8n-nodes-base.if | Validation gate | Enrich CSAT Data | Determine CSAT Channel | ## Fetch completed issue... |
| Parse CSAT Webhook Data | n8n-nodes-base.code | Webhook parser | Configure CSAT Form Path | Verify CSAT Webhook | ## Parse CSAT webhook... |
| Verify Webhook Token | n8n-nodes-base.if | Security check | CSAT Webhook Listener | Configure CSAT Form Path | ## CSAT webhook intake... |
| CSAT Webhook Listener | n8n-nodes-base.webhook | Webhook trigger | — | Verify Webhook Token | ## CSAT webhook intake... |
| Label Jira Auto Reply Sent | n8n-nodes-base.httpRequest | Jira REST API client | Comment Jira Auto Reply | — | ## Mark auto-reply sent... |
| Comment Jira Auto Reply | n8n-nodes-base.httpRequest | Jira REST API client | Prepare Auto Reply Mark | Label Jira Auto Reply Sent | ## Mark auto-reply sent... |
| Prepare Auto Reply Mark | n8n-nodes-base.code | Payload builder | Associate Note in HubSpot, Check HubSpot Note Eligibility | Comment Jira Auto Reply | ## Mark auto-reply sent... |
| Associate Note in HubSpot | n8n-nodes-base.httpRequest | CRM API client | Prepare HubSpot Association | Prepare Auto Reply Mark | ## Create reply note... |
| Prepare HubSpot Association | n8n-nodes-base.code | Payload mapper | Create Note in HubSpot | Associate Note in HubSpot | ## Create reply note... |
| Create Note in HubSpot | n8n-nodes-base.httpRequest | CRM API client | Build Note Data for HubSpot | Prepare HubSpot Association | ## Create reply note... |
| Build Note Data for HubSpot | n8n-nodes-base.code | Payload builder | Check HubSpot Note Eligibility | Create Note in HubSpot | ## Prepare HubSpot logging... |
| Check HubSpot Note Eligibility | n8n-nodes-base.if | CRM gating router | Send Auto Reply via WhatsApp, Email Auto Reply via Gmail | Build Note Data for HubSpot, Prepare Auto Reply Mark | ## Prepare HubSpot logging... |
| Send Auto Reply via WhatsApp | n8n-nodes-base.httpRequest | WAHA API client | Check WhatsApp Channel | Check HubSpot Note Eligibility | ## Select reply channel... |
| Email Auto Reply via Gmail | n8n-nodes-base.gmail | Gmail integration | Check WhatsApp Channel | Check HubSpot Note Eligibility | ## Select reply channel... |
| Check WhatsApp Channel | n8n-nodes-base.if | Channel router | Evaluate Auto Reply Conditions | Send Auto Reply via WhatsApp, Email Auto Reply via Gmail | ## Select reply channel... |
| Evaluate Auto Reply Conditions | n8n-nodes-base.if | Validation gate | Normalize Auto Reply Payload | Check WhatsApp Channel | ## Normalize auto-reply payload... |
| Normalize Auto Reply Payload | n8n-nodes-base.code | Payload normalization | Load Auto Reply Config | Evaluate Auto Reply Conditions | ## Normalize auto-reply payload... |
| Load Auto Reply Config | n8n-nodes-base.code | Configuration loader | Trigger Workflow Execution | Normalize Auto Reply Payload | ## Close-loop entry config... |
| Trigger Workflow Execution | n8n-nodes-base.executeWorkflowTrigger | Sub-workflow trigger | — | Load Auto Reply Config | ## Close-loop entry config... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Entry Points Setup
1. Create an **Execute Workflow Trigger** (`Trigger Workflow Execution`) node to receive payloads from parent workflows.
2. Create a **Webhook** (`CSAT Webhook Listener`) node on HTTP method `POST` at path `jira-csat`.
3. Create a **Form Trigger** (`CSAT Form Submission Trigger`) node configured with dropdown choices (`1` to `5`) for the score and a textarea for comments.

#### Step 2: Auto-Reply Path Configuration
1. Connect `Trigger Workflow Execution` to a **Code** node (`Load Auto Reply Config`) defining the `CLOSE_LOOP` constant containing your Jira base URL, project key, custom field IDs, and label settings.
2. Connect to a **Code** node (`Normalize Auto Reply Payload`) to sanitize incoming contact channels and build WhatsApp chat IDs.
3. Add an **IF** node (`Evaluate Auto Reply Conditions`) verifying `{{ $json.canSend }}`.
4. Add an **IF** node (`Check WhatsApp Channel`) checking if `{{ $json.channel }}` equals `"whatsapp"`.
5. Branch True to an **HTTP Request** node (`Send Auto Reply via WhatsApp`) pointing to your WAHA endpoint (`/api/sendText`) using HTTP Header Auth.
6. Branch False to a **Gmail** node (`Email Auto Reply via Gmail`) using OAuth2 credentials.
7. Route messages through an **IF** node (`Check HubSpot Note Eligibility`) checking for `hubspotContactId`.
8. If eligible, add a **Code** node (`Build Note Data for HubSpot`), an **HTTP Request** (`Create Note in HubSpot`), a **Code** node (`Prepare HubSpot Association`), and an **HTTP Request** (`Associate Note in HubSpot`) targeting HubSpot API endpoints.
9. Finish the path with a **Code** node (`Prepare Auto Reply Mark`), an **HTTP Request** (`Comment Jira Auto Reply`), and an **HTTP Request** (`Label Jira Auto Reply Sent`) using Jira HTTP Basic Auth.

#### Step 3: CSAT Webhook & Survey Dispatch Path Setup
1. Connect `CSAT Webhook Listener` to an **IF** node (`Verify Webhook Token`) validating `token` query parameters against your secret.
2. Connect to a **Code** node (`Configure CSAT Form Path`) injecting the active form webhook path UUID.
3. Connect to a **Code** node (`Parse CSAT Webhook Data`) to verify issue status changes to "Done".
4. Add an **IF** node (`Verify CSAT Webhook`) checking `{{ $json.canProceed }}`.
5. Add a **Code** node (`Build CSAT Fetch Query`) and an **HTTP Request** (`Fetch CSAT Done Issue from Jira`) to retrieve issue details.
6. Enrich data via a **Code** node (`Enrich CSAT Data`) building survey URLs.
7. Validate via an **IF** node (`Evaluate CSAT Sending Criteria`), then route via an **IF** node (`Determine CSAT Channel`) to either **WhatsApp** (`Send CSAT via WhatsApp`) or **Gmail** (`Email CSAT via Gmail`).
8. Record success in Jira using a **Code** node (`Prepare CSAT Sent Mark`), an **HTTP Request** (`Comment Jira CSAT Sent`), and an **HTTP Request** (`Label Jira Issue CSAT Sent`).

#### Step 4: CSAT Form Submission & CRM Sync Path Setup
1. Connect `CSAT Form Submission Trigger` to a **Code** node (`Normalize CSAT Form Data`) validating ticket key formatting and score limits.
2. Retrieve and enrich issue data using a **Code** (`Retrieve Jira CSAT Issue`), **HTTP Request** (`Fetch CSAT Issue from Jira`), and **Code** (`Extract CSAT Details from Jira`).
3. Format and save comments in Jira via a **Code** (`Prepare CSAT for Saving`) and **HTTP Request** (`Post CSAT Score Comment`).
4. Check custom field configuration via an **IF** node (`Verify CSAT Custom Field`) and update scores via an **HTTP Request** (`Update Jira CSAT Score`).
5. Check HubSpot eligibility via an **IF** node (`Check HubSpot CSAT Eligibility`). When valid, create and associate notes using **Code** and **HTTP Request** nodes (`Prepare HubSpot Note Data`, `Create CSAT Note on HubSpot`, `Prepare HubSpot Association Data`, `Post CSAT Note Association`).
6. Conclude execution by attaching the **Form** (`Capture CSAT Form Response`) completion node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Website | https://www.intuz.com/n8n-workflow-automation-templates/ |
| Email | getstarted@intuz.com |
| LinkedIn | https://www.linkedin.com/company/intuz |
| Get Started | https://n8n.partnerlinks.io/intuz |
| Custom Workflow Automation | https://www.intuz.com/get-started/ |