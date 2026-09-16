Route multichannel support tickets with OpenAI, HubSpot, Jira, and Slack

https://n8nworkflows.xyz/workflows/route-multichannel-support-tickets-with-openai--hubspot--jira--and-slack-18187


# Route multichannel support tickets with OpenAI, HubSpot, Jira, and Slack

### 1. Workflow Overview

This workflow automates multichannel support ticket management by ingesting requests from Gmail, WhatsApp (via WAHA webhook), and a website contact form. It normalizes inputs, enriches customer profiles using HubSpot CRM, classifies urgency and intent via OpenAI, maps tickets to appropriate teams and calculates SLAs, manages deduplication in Jira, posts notifications to Slack, and runs periodic SLA breach monitoring.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Ingests events from Gmail, WhatsApp, and the n8n Contact Form, filters payloads, and normalizes them into a unified ticket structure while preserving binary attachments.
- **1.2 CRM Enrichment & AI Triage:** Loads configuration, queries HubSpot CRM for customer context, filters text, and uses OpenAI to analyze category, sentiment, urgency, and summary.
- **1.3 SLA, Routing & Jira Processing:** Computes SLA deadlines, generates Jira payloads, searches for duplicates, creates or flags issues, stamps SLA metadata, and uploads attachments.
- **1.4 Notifications & Completion:** Posts alerts/confirmations to Slack and responds to form submitters.
- **1.5 Periodic SLA Breach Monitoring:** Runs a 10-minute schedule to catch unresolved tickets past their SLA deadline, alert Slack/managers, escalate priority, and update ticket states.
- **1.6 Error Handling:** Catches global errors and routes notifications to a dedicated Slack error channel.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Normalization
- **Overview:** Ingests inbound requests from three distinct channels (Gmail, WhatsApp, and Web Form), normalizes their schemas into a unified ticket structure, handles conditional media downloads for WhatsApp messages, and holds binary data for downstream attachment processing.
- **Nodes Involved:** 
  - `When Gmail Received`
  - `Normalize Gmail Data`
  - `WAHA Support Webhook`
  - `Filter WAHA Messages`
  - `Parse WAHA Data`
  - `Check for Media`
  - `Fetch WAHA Media`
  - `Restore WAHA Media Data`
  - `Normalize WhatsApp Data`
  - `When Form Submitted`
  - `Normalize Form Data`
  - `Hold Binary For Upload`
- **Node Details:**
  - **When Gmail Received**
    - *Type and technical role:* `n8n-nodes-base.gmailTrigger` (Trigger)
    - *Configuration choices:* Polls the connected inbox for new support emails.
    - *Key expressions or variables:* Default trigger parameters.
    - *Input/Output:* Output connected to `Normalize Gmail Data`.
    - *Edge cases:* Authentication token expiration or rate limits during polling.
  - **Normalize Gmail Data**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* JavaScript execution extracting sender name, email address, subject, and body.
    - *Input/Output:* Input from `When Gmail Received`, output connected to `Hold Binary For Upload`.
  - **WAHA Support Webhook**
    - *Type and technical role:* `n8n-nodes-base.webhook` (Webhook Trigger)
    - *Configuration choices:* Listens for incoming WhatsApp events on webhook ID `waha-support-webhook`.
    - *Input/Output:* Output connected to `Filter WAHA Messages`.
  - **Filter WAHA Messages**
    - *Type and technical role:* `n8n-nodes-base.code` (Filtering)
    - *Configuration choices:* Filters out unwanted event types (e.g., outgoing or status messages).
    - *Input/Output:* Input from `WAHA Support Webhook`, output connected to `Parse WAHA Data`.
  - **Parse WAHA Data**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Extracts phone number, message text, and media metadata.
    - *Input/Output:* Input from `Filter WAHA Messages`, output connected to `Check for Media`.
  - **Check for Media**
    - *Type and technical role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration choices:* Evaluates whether the inbound WhatsApp payload contains media files.
    - *Input/Output:* Input from `Parse WAHA Data`. True branch connected to `Fetch WAHA Media`; False branch connected to `Normalize WhatsApp Data`.
  - **Fetch WAHA Media**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* Calls WAHA `/api/files` endpoint using `X-Api-Key` header auth with retry settings (`onError: continueRegularOutput`, max 3 tries).
    - *Input/Output:* Input from `Check for Media` (True), output connected to `Restore WAHA Media Data`.
    - *Edge cases:* Returns HTTP 401 if the `WAHA Header Auth` credential is missing or invalid.
  - **Restore WAHA Media Data**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Merges fetched media binary data back into the message payload context.
    - *Input/Output:* Input from `Fetch WAHA Media`, output connected to `Normalize WhatsApp Data`.
  - **Normalize WhatsApp Data**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Maps WhatsApp message properties to the common ticket schema.
    - *Input/Output:* Inputs from `Check for Media` (False) or `Restore WAHA Media Data`, output connected to `Hold Binary For Upload`.
  - **When Form Submitted**
    - *Type and technical role:* `n8n-nodes-base.formTrigger` (Form Trigger)
    - *Configuration choices:* Receives website contact form submits on webhook ID `8e4fd1e6-bb88-49b8-ac84-ccd8bf133513`.
    - *Input/Output:* Output connected to `Normalize Form Data`.
  - **Normalize Form Data**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Maps contact form fields into the common ticket format.
    - *Input/Output:* Input from `When Form Submitted`, output connected to `Hold Binary For Upload`.
  - **Hold Binary For Upload**
    - *Type and technical role:* `n8n-nodes-base.noOp` (Pass-through / Stash)
    - *Configuration choices:* Acts as an in-line file stash point bridging normalized inputs to CRM configuration.
    - *Input/Output:* Inputs from `Normalize Gmail Data`, `Normalize WhatsApp Data`, and `Normalize Form Data`; output connected to `Load Configuration`.

#### Block 1.2: CRM Enrichment & AI Triage
- **Overview:** Injects environment/ticket configuration, queries HubSpot CRM to locate or merge customer records based on email/phone, prepares raw text, and executes an OpenAI language model analysis to categorize and prioritize the support request.
- **Nodes Involved:**
  - `Load Configuration`
  - `Prepare HubSpot Search Query`
  - `Post HubSpot Search`
  - `Consolidate HubSpot Results`
  - `Filter Text for OpenAI`
  - `OpenAI Analysis`
  - `Decode AI Analysis`
- **Node Details:**
  - **Load Configuration**
    - *Type and technical role:* `n8n-nodes-base.code` (Initialization)
    - *Configuration choices:* Loads hardcoded operational configurations (Jira base URLs, project keys, assignee IDs, Slack channels).
    - *Input/Output:* Input from `Hold Binary For Upload`, output connected to `Prepare HubSpot Search Query`.
  - **Prepare HubSpot Search Query**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Formats a search payload using requester email or phone for HubSpot lookup.
    - *Input/Output:* Input from `Load Configuration`, output connected to `Post HubSpot Search`.
  - **Post HubSpot Search**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* Executes a POST request to HubSpot private app endpoints using header authentication (`continueOnFail: true`, max 3 tries).
    - *Input/Output:* Input from `Prepare HubSpot Search Query`, output connected to `Consolidate HubSpot Results`.
    - *Edge cases:* API throttling or invalid private app tokens.
  - **Consolidate HubSpot Results**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Parses HubSpot response, merging found CRM customer properties into the ticket context.
    - *Input/Output:* Input from `Post HubSpot Search`, output connected to `Filter Text for OpenAI`.
  - **Filter Text for OpenAI**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Cleans and truncates ticket body text to prepare safe prompt inputs for OpenAI.
    - *Input/Output:* Input from `Consolidate HubSpot Results`, output connected to `OpenAI Analysis`.
  - **OpenAI Analysis**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.openAi` (AI Model Execution)
    - *Configuration choices:* Calls OpenAI model to classify ticket category, urgency, sentiment, generate a summary, and suggest a reply (max 3 tries).
    - *Input/Output:* Input from `Filter Text for OpenAI`, output connected to `Decode AI Analysis`.
    - *Edge cases:* OpenAI rate limits, context window overflow, or non-JSON output structures.
  - **Decode AI Analysis**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Parses the model response into clean, structured JSON properties.
    - *Input/Output:* Input from `OpenAI Analysis`, output connected to `Compute SLA and Team Allocation`.

#### Block 1.3: SLA, Routing & Jira Processing
- **Overview:** Calculates SLA response deadlines, maps ticket categories to team allocations, formats and checks Jira for duplicates, creates new issues or flags duplicates, and stamps custom metadata and attachments.
- **Nodes Involved:**
  - `Compute SLA and Team Allocation`
  - `Generate Jira Payload`
  - `Search Jira for Duplicates`
  - `Parse Deduplication Result`
  - `Identify Duplicate Issues`
  - `Create Jira Issue4`
  - `Merge Jira Create Result4`
  - `Prepare SLA Stamp3`
  - `Should Stamp SLA?3`
  - `Stamp SLA Custom Fields4`
  - `Restore After SLA Stamp3`
  - `Prepare Attachments4`
  - `Skip Attachments?4`
  - `Upload Jira Attachment4`
  - `Restore After Upload3`
- **Node Details:**
  - **Compute SLA and Team Allocation**
    - *Type and technical role:* `n8n-nodes-base.code` (Business Logic)
    - *Configuration choices:* Calculates SLA due times based on priority and assigns the owning team.
    - *Input/Output:* Input from `Decode AI Analysis`, output connected to `Generate Jira Payload`.
  - **Generate Jira Payload**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Constructs the target Jira issue payload (summary, description, custom fields, assignee).
    - *Input/Output:* Input from `Compute SLA and Team Allocation`, output connected to `Search Jira for Duplicates`.
  - **Search Jira for Duplicates**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* Queries Jira Cloud using JQL to search for issues matching the computed deduplication key.
    - *Input/Output:* Input from `Generate Jira Payload`, output connected to `Parse Deduplication Result`.
  - **Parse Deduplication Result**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Evaluates search counts to determine issue existence.
    - *Input/Output:* Input from `Search Jira for Duplicates`, output connected to `Identify Duplicate Issues`.
  - **Identify Duplicate Issues**
    - *Type and technical role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration choices:* Branches execution depending on whether a duplicate issue was found.
    - *Input/Output:* Input from `Parse Deduplication Result`. True branch connected to `Build Duplicated Result`; False branch connected to `Create Jira Issue4`.
  - **Create Jira Issue4**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* POST request to Jira REST API to create a new support ticket (max 3 tries).
    - *Input/Output:* Input from `Identify Duplicate Issues` (False), output connected to `Merge Jira Create Result4`.
  - **Merge Jira Create Result4**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Consolidates Jira issue creation output data.
    - *Input/Output:* Input from `Create Jira Issue4`, output connected to `Prepare SLA Stamp3`.
  - **Prepare SLA Stamp3**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Prepares field values for SLA metadata stamping.
    - *Input/Output:* Input from `Merge Jira Create Result4`, output connected to `Should Stamp SLA?3`.
  - **Should Stamp SLA?3**
    - *Type and technical role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration choices:* Determines whether custom SLA fields require updating on the Jira issue.
    - *Input/Output:* Input from `Prepare SLA Stamp3`. True branch connected to `Stamp SLA Custom Fields4`; False branch connected to `Prepare Attachments4`.
  - **Stamp SLA Custom Fields4**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* PUT request to update Jira custom fields with SLA due dates.
    - *Input/Output:* Input from `Should Stamp SLA?3` (True), output connected to `Restore After SLA Stamp3`.
  - **Restore After SLA Stamp3**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Restores ticket context after SLA field update.
    - *Input/Output:* Input from `Stamp SLA Custom Fields4`, output connected to `Prepare Attachments4`.
  - **Prepare Attachments4**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Inspects ticket context for binary attachment arrays.
    - *Input/Output:* Inputs from `Should Stamp SLA?3` (False) or `Restore After SLA Stamp3`, output connected to `Skip Attachments?4`.
  - **Skip Attachments?4**
    - *Type and technical role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration choices:* Checks if attachments are present for upload.
    - *Input/Output:* Input from `Prepare Attachments4`. True branch connected to `Normalize Before Slack3`; False branch connected to `Upload Jira Attachment4`.
  - **Upload Jira Attachment4**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* Uploads attached binary files to the newly created Jira issue.
    - *Input/Output:* Input from `Skip Attachments?4` (False), output connected to `Restore After Upload3`.
  - **Restore After Upload3**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Restores ticket context following file upload.
    - *Input/Output:* Input from `Upload Jira Attachment4`, output connected to `Normalize Before Slack3`.

#### Block 1.4: Notifications & Completion
- **Overview:** Formats success outputs, posts notifications to appropriate Slack channels (new tickets, duplicates), and completes the response cycle for web contact form submissions.
- **Nodes Involved:**
  - `Build Duplicated Result`
  - `Notify Slack of Duplicate`
  - `Normalize Before Slack3`
  - `Slack Ticket Notification4`
  - `Build Primary Result`
  - `Merge Ticket with Slack`
  - `Merge All Results`
  - `Is Form Channel?`
  - `Reply Form Success`
- **Node Details:**
  - **Build Duplicated Result**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Formats payload for duplicate ticket notices.
    - *Input/Output:* Input from `Identify Duplicate Issues` (True), output connected to `Notify Slack of Duplicate`.
  - **Notify Slack of Duplicate**
    - *Type and technical role:* `n8n-nodes-base.slack` (Chat Integration)
    - *Configuration choices:* Posts a duplicate issue notification message to Slack.
    - *Input/Output:* Input from `Build Duplicated Result`, output connected to `Merge All Results`.
  - **Normalize Before Slack3**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Prepares message formatting for Slack ticket notifications.
    - *Input/Output:* Inputs from `Skip Attachments?4` (True) or `Restore After Upload3`, outputs connected in parallel to `Build Primary Result` and `Slack Ticket Notification4`.
  - **Slack Ticket Notification4**
    - *Type and technical role:* `n8n-nodes-base.slack` (Chat Integration)
    - *Configuration choices:* Posts new ticket alert details to the triage Slack channel (`webhookId: 924028dd-afcc-4e1f-aae7-c6c2cd664f33`).
    - *Input/Output:* Input from `Normalize Before Slack3`, output connected to `Merge Ticket with Slack` (index 1).
  - **Build Primary Result**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Constructs primary success result object.
    - *Input/Output:* Input from `Normalize Before Slack3`, output connected to `Merge Ticket with Slack`.
  - **Merge Ticket with Slack**
    - *Type and technical role:* `n8n-nodes-base.merge` (Data Merger)
    - *Configuration choices:* Combines primary results and Slack output confirmations.
    - *Input/Output:* Inputs from `Build Primary Result` and `Slack Ticket Notification4`, output connected to `Merge All Results`.
  - **Merge All Results**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Merges duplicated and primary ticket processing paths into a unified stream.
    - *Input/Output:* Inputs from `Notify Slack of Duplicate` and `Merge Ticket with Slack`, output connected to `Is Form Channel?`.
  - **Is Form Channel?**
    - *Type and technical role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration choices:* Checks if the originating request came from a web contact form.
    - *Input/Output:* Input from `Merge All Results`. True branch connected to `Reply Form Success`.
  - **Reply Form Success**
    - *Type and technical role:* `n8n-nodes-base.form` (Form Response)
    - *Configuration choices:* Returns confirmation message back to the website form submitter.
    - *Input/Output:* Input from `Is Form Channel?` (True).

#### Block 1.5: Periodic SLA Breach Monitoring
- **Overview:** Runs every 10 minutes to search Jira for unresolved tickets past their SLA deadline, posts alerts to Slack and manager DMs, escalates priority if configured, and marks tickets as notified to prevent duplicate alerts.
- **Nodes Involved:**
  - `Every 10 Minutes1`
  - `Load SLA Config1`
  - `Search Breached Tickets1`
  - `Split Breached Issues1`
  - `Slack SLA Alert1`
  - `Has Manager?1`
  - `Notify Manager1`
  - `Raise Priority?1`
  - `Raise Jira Priority1`
  - `Restore Before Mark1`
  - `Pass Ticket (No Raise)`
  - `Mark SLA Notified1`
  - `Summarize Mark Result`
- **Node Details:**
  - **Every 10 Minutes1**
    - *Type and technical role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration choices:* Triggers execution every 10 minutes.
    - *Input/Output:* Output connected to `Load SLA Config1`.
  - **Load SLA Config1**
    - *Type and technical role:* `n8n-nodes-base.code` (Initialization)
    - *Configuration choices:* Loads constants for SLA breach searches.
    - *Input/Output:* Input from `Every 10 Minutes1`, output connected to `Search Breached Tickets1`.
  - **Search Breached Tickets1**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* Queries Jira for open issues where SLA due time has passed and notification flag is false.
    - *Input/Output:* Input from `Load SLA Config1`, output connected to `Split Breached Issues1`.
  - **Split Breached Issues1**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Splits array of breached issues into individual processing items.
    - *Input/Output:* Input from `Search Breached Tickets1`, outputs connected to `Slack SLA Alert1`, `Has Manager?1`, and `Raise Priority?1`.
  - **Slack SLA Alert1**
    - *Type and technical role:* `n8n-nodes-base.slack` (Chat Integration)
    - *Configuration choices:* Posts SLA breach warning to the configured Slack channel.
    - *Input/Output:* Input from `Split Breached Issues1`.
  - **Has Manager?1**
    - *Type and technical role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration choices:* Checks if a manager ID is recorded for escalation.
    - *Input/Output:* Input from `Split Breached Issues1`. True branch connected to `Notify Manager1`.
  - **Notify Manager1**
    - *Type and technical role:* `n8n-nodes-base.slack` (Chat Integration)
    - *Configuration choices:* Sends direct message alert to manager regarding the breach.
    - *Input/Output:* Input from `Has Manager?1` (True).
  - **Raise Priority?1**
    - *Type and technical role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration choices:* Evaluates whether to escalate the Jira issue priority.
    - *Input/Output:* Input from `Split Breached Issues1`. True branch connected to `Raise Jira Priority1`; False branch connected to `Pass Ticket (No Raise)`.
  - **Raise Jira Priority1**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* Updates Jira issue priority via API PUT request.
    - *Input/Output:* Input from `Raise Priority?1` (True), output connected to `Restore Before Mark1`.
  - **Restore Before Mark1**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Restores ticket context after priority escalation.
    - *Input/Output:* Input from `Raise Jira Priority1`, output connected to `Mark SLA Notified1`.
  - **Pass Ticket (No Raise)**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Passes ticket through without priority adjustment.
    - *Input/Output:* Input from `Raise Priority?1` (False), output connected to `Mark SLA Notified1`.
  - **Mark SLA Notified1**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration choices:* Updates Jira ticket custom field to mark the SLA breach as notified.
    - *Input/Output:* Inputs from `Restore Before Mark1` or `Pass Ticket (No Raise)`, output connected to `Summarize Mark Result`.
  - **Summarize Mark Result**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Summarizes breach processing results.
    - *Input/Output:* Input from `Mark SLA Notified1`.

#### Block 1.6: Error Handling
- **Overview:** Catches unhandled exceptions from any workflow node and formats an error payload for delivery to a dedicated Slack error channel.
- **Nodes Involved:**
  - `Error Trigger`
  - `Format Error Payload`
  - `Slack Error Channel`
- **Node Details:**
  - **Error Trigger**
    - *Type and technical role:* `n8n-nodes-base.errorTrigger` (Trigger)
    - *Configuration choices:* Listens for failure events across all nodes in the workflow.
    - *Input/Output:* Output connected to `Format Error Payload`.
  - **Format Error Payload**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Extracts error messages, node names, execution IDs, and timestamps.
    - *Input/Output:* Input from `Error Trigger`, output connected to `Slack Error Channel`.
  - **Slack Error Channel**
    - *Type and technical role:* `n8n-nodes-base.slack` (Chat Integration)
    - *Configuration choices:* Posts formatted error details to the operations/error Slack channel (`webhookId: 451718ba-c741-41ba-bbd6-aad0a4e0019d`).
    - *Input/Output:* Input from `Format Error Payload`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note11 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note12 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note13 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note14 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note15 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note16 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note17 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note18 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note19 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note20 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note21 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note22 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note23 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note24 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| Sticky Note25 | n8n-nodes-base.stickyNote | Visual container | — | — |  |
| When Gmail Received | n8n-nodes-base.gmailTrigger | Polls inbox for new support emails | — | Normalize Gmail Data | Polls inbox for new support emails |
| Normalize Gmail Data | n8n-nodes-base.code | Extracts name, email, subject & body | When Gmail Received | Hold Binary For Upload |  |
| WAHA Support Webhook | n8n-nodes-base.webhook | Receives inbound WhatsApp events | — | Filter WAHA Messages | Receives inbound WhatsApp events |
| Filter WAHA Messages | n8n-nodes-base.code | Filters unwanted WAHA events | WAHA Support Webhook | Parse WAHA Data |  |
| Parse WAHA Data | n8n-nodes-base.code | Extracts phone, message & media | Filter WAHA Messages | Check for Media | Extracts phone, message & media |
| Check for Media | n8n-nodes-base.if | Checks if media is present in WAHA payload | Parse WAHA Data | Fetch WAHA Media, Normalize WhatsApp Data |  |
| Fetch WAHA Media | n8n-nodes-base.httpRequest | Fetches WhatsApp media files | Check for Media | Restore WAHA Media Data | WAHA /api/files needs header X-Api-Key. Select credential 'WAHA Header Auth' (Header Auth, Name=X-Api-Key, Value=plain WAHA key) on this node after import. 401 here means the credential is not selected or the key is wrong. |
| Normalize WhatsApp Data | n8n-nodes-base.code | Maps WhatsApp data to common format | Check for Media, Restore WAHA Media Data | Hold Binary For Upload | Maps WhatsApp data to common format |
| Restore WAHA Media Data | n8n-nodes-base.code | Restores media binary context | Fetch WAHA Media | Normalize WhatsApp Data |  |
| When Form Submitted | n8n-nodes-base.formTrigger | Receives website contact form submits | — | Normalize Form Data | Receives website contact form submits |
| Normalize Form Data | n8n-nodes-base.code | Maps form data to common format | When Form Submitted | Hold Binary For Upload | Maps form data to common format |
| Reply Form Success | n8n-nodes-base.form | Confirms receipt to the website form | Is Form Channel? | — | Confirms receipt to the website form |
| Merge All Results | n8n-nodes-base.code | Merges processed ticket results | Notify Slack of Duplicate, Merge Ticket with Slack | Is Form Channel? |  |
| Build Primary Result | n8n-nodes-base.code | Builds primary ticket result | Normalize Before Slack3 | Merge Ticket with Slack |  |
| Build Duplicated Result | n8n-nodes-base.code | Builds duplicate ticket notice payload | Identify Duplicate Issues | Notify Slack of Duplicate |  |
| Merge Ticket with Slack | n8n-nodes-base.merge | Merges ticket data with Slack notifications | Build Primary Result, Slack Ticket Notification4 | Merge All Results |  |
| Load Configuration | n8n-nodes-base.code | Loads environment configuration | Hold Binary For Upload | Prepare HubSpot Search Query |  |
| Prepare HubSpot Search Query | n8n-nodes-base.code | Formats HubSpot search parameters | Load Configuration | Post HubSpot Search |  |
| Post HubSpot Search | n8n-nodes-base.httpRequest | Looks up requester in HubSpot | Prepare HubSpot Search Query | Consolidate HubSpot Results | Looks up requester in HubSpot |
| Consolidate HubSpot Results | n8n-nodes-base.code | Merges HubSpot customer context | Post HubSpot Search | Filter Text for OpenAI |  |
| Filter Text for OpenAI | n8n-nodes-base.code | Cleans text for OpenAI analysis | Consolidate HubSpot Results | OpenAI Analysis |  |
| OpenAI Analysis | @n8n/n8n-nodes-langchain.openAi | AI classifies category, urgency & sentiment | Filter Text for OpenAI | Decode AI Analysis | AI classifies category, urgency & sentiment |
| Decode AI Analysis | n8n-nodes-base.code | Parses AI response into structured fields | OpenAI Analysis | Compute SLA and Team Allocation | Parses AI response into structured fields |
| Compute SLA and Team Allocation | n8n-nodes-base.code | Assigns SLA deadline & owning team | Decode AI Analysis | Generate Jira Payload | Assigns SLA deadline & owning team |
| Generate Jira Payload | n8n-nodes-base.code | Prepares Jira issue payload | Compute SLA and Team Allocation | Search Jira for Duplicates |  |
| Search Jira for Duplicates | n8n-nodes-base.httpRequest | Checks Jira for an existing ticket | Generate Jira Payload | Parse Deduplication Result | Checks Jira for an existing ticket |
| Parse Deduplication Result | n8n-nodes-base.code | Parses deduplication check response | Search Jira for Duplicates | Identify Duplicate Issues |  |
| Identify Duplicate Issues | n8n-nodes-base.if | Branches duplicate vs new ticket | Parse Deduplication Result | Build Duplicated Result, Create Jira Issue4 | Branches duplicate vs new ticket |
| Notify Slack of Duplicate | n8n-nodes-base.slack | Posts duplicate notice to Slack | Build Duplicated Result | Merge All Results |  |
| Create Jira Issue4 | n8n-nodes-base.httpRequest | Creates the Jira support ticket | Identify Duplicate Issues | Merge Jira Create Result4 | Creates the Jira support ticket |
| Merge Jira Create Result4 | n8n-nodes-base.code | Merges Jira creation results | Create Jira Issue4 | Prepare SLA Stamp3 |  |
| Stamp SLA Custom Fields4 | n8n-nodes-base.httpRequest | Writes SLA due date to Jira | Should Stamp SLA?3 | Restore After SLA Stamp3 | Writes SLA due date to Jira |
| Prepare Attachments4 | n8n-nodes-base.code | Prepares customer attachments for upload | Should Stamp SLA?3, Restore After SLA Stamp3 | Skip Attachments?4 |  |
| Skip Attachments?4 | n8n-nodes-base.if | Decides whether to skip attachments | Prepare Attachments4 | Normalize Before Slack3, Upload Jira Attachment4 |  |
| Upload Jira Attachment4 | n8n-nodes-base.httpRequest | Attaches customer files to the ticket | Skip Attachments?4 | Restore After Upload3 | Attaches customer files to the ticket |
| Normalize Before Slack3 | n8n-nodes-base.code | Normalizes ticket data before Slack posting | Skip Attachments?4, Restore After Upload3 | Build Primary Result, Slack Ticket Notification4 |  |
| Slack Ticket Notification4 | n8n-nodes-base.slack | Posts new ticket alert to Slack | Normalize Before Slack3 | Merge Ticket with Slack | Posts new ticket alert to Slack |
| Prepare SLA Stamp3 | n8n-nodes-base.code | Prepares SLA stamp configuration | Merge Jira Create Result4 | Should Stamp SLA?3 |  |
| Should Stamp SLA?3 | n8n-nodes-base.if | Decides if SLA fields need stamping | Prepare SLA Stamp3 | Stamp SLA Custom Fields4, Prepare Attachments4 | Decides if SLA fields need stamping |
| Restore After SLA Stamp3 | n8n-nodes-base.code | Restores context after SLA stamp | Stamp SLA Custom Fields4 | Prepare Attachments4 |  |
| Restore After Upload3 | n8n-nodes-base.code | Restores context after file upload | Upload Jira Attachment4 | Normalize Before Slack3 |  |
| Pass Ticket (No Raise) | n8n-nodes-base.code | Passes ticket without priority change | Raise Priority?1 | Mark SLA Notified1 |  |
| Summarize Mark Result | n8n-nodes-base.code | Summarizes SLA mark result | Mark SLA Notified1 | — |  |
| Every 10 Minutes1 | n8n-nodes-base.scheduleTrigger | Runs the SLA breach check on schedule | — | Load SLA Config1 | Runs the SLA breach check on schedule |
| Load SLA Config1 | n8n-nodes-base.code | Loads SLA breach check constants | Every 10 Minutes1 | Search Breached Tickets1 |  |
| Search Breached Tickets1 | n8n-nodes-base.httpRequest | Finds tickets past their SLA deadline | Load SLA Config1 | Split Breached Issues1 | Finds tickets past their SLA deadline |
| Split Breached Issues1 | n8n-nodes-base.code | Splits breached issues array | Search Breached Tickets1 | Slack SLA Alert1, Has Manager?1, Raise Priority?1 |  |
| Slack SLA Alert1 | n8n-nodes-base.slack | Posts SLA breach warning to Slack | Split Breached Issues1 | — | Posts SLA breach warning to Slack |
| Has Manager?1 | n8n-nodes-base.if | Checks if a manager is on record | Split Breached Issues1 | Notify Manager1 | Checks if a manager is on record |
| Notify Manager1 | n8n-nodes-base.slack | Alerts the manager of the breach | Has Manager?1 | — | Alerts the manager of the breach |
| Raise Priority?1 | n8n-nodes-base.if | Decides whether to escalate priority | Split Breached Issues1 | Raise Jira Priority1, Pass Ticket (No Raise) | Decides whether to escalate priority |
| Raise Jira Priority1 | n8n-nodes-base.httpRequest | Bumps ticket priority in Jira | Raise Priority?1 | Restore Before Mark1 | Bumps ticket priority in Jira |
| Restore Before Mark1 | n8n-nodes-base.code | Restores context before marking SLA | Raise Jira Priority1 | Mark SLA Notified1 |  |
| Mark SLA Notified1 | n8n-nodes-base.httpRequest | Flags ticket so it isn't re-alerted | Restore Before Mark1, Pass Ticket (No Raise) | Summarize Mark Result | Flags ticket so it isn't re-alerted |
| Error Trigger | n8n-nodes-base.errorTrigger | Catches failures from any node | — | Format Error Payload | Catches failures from any node |
| Format Error Payload | n8n-nodes-base.code | Formats error event payload | Error Trigger | Slack Error Channel |  |
| Slack Error Channel | n8n-nodes-base.slack | Reports workflow errors to Slack | Format Error Payload | — | Reports workflow errors to Slack |
| Is Form Channel? | n8n-nodes-base.if | Routes form submissions to a reply | Merge All Results | Reply Form Success | Routes form submissions to a reply |
| Hold Binary For Upload | n8n-nodes-base.noOp | In-line binary file stash | Normalize Gmail Data, Normalize WhatsApp Data, Normalize Form Data | Load Configuration | In-line file stash (old sub-workflow trigger). Do not fan this out as a dead-end. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow in n8n manually:

1. **Create the Triggers and Ingestion Nodes:**
   - Add **When Gmail Received** (`n8n-nodes-base.gmailTrigger`) and connect it to a **Normalize Gmail Data** Code node.
   - Add **WAHA Support Webhook** (`n8n-nodes-base.webhook`, webhook ID `waha-support-webhook`) -> **Filter WAHA Messages** Code node -> **Parse WAHA Data** Code node -> **Check for Media** IF node.
   - For **Check for Media** (True branch): Add **Fetch WAHA Media** HTTP Request node (configure with `X-Api-Key` Header Auth credential, target `/api/files`) -> **Restore WAHA Media Data** Code node -> **Normalize WhatsApp Data** Code node.
   - For **Check for Media** (False branch): Connect directly to **Normalize WhatsApp Data** Code node.
   - Add **When Form Submitted** (`n8n-nodes-base.formTrigger`, webhook ID `8e4fd1e6-bb88-49b8-ac84-ccd8bf133513`) -> **Normalize Form Data** Code node.
   - Connect **Normalize Gmail Data**, **Normalize WhatsApp Data**, and **Normalize Form Data** into a **Hold Binary For Upload** No-Op node.

2. **Configure CRM Enrichment and AI Processing:**
   - Connect **Hold Binary For Upload** to **Load Configuration** Code node -> **Prepare HubSpot Search Query** Code node -> **Post HubSpot Search** HTTP Request node (using HubSpot private app auth) -> **Consolidate HubSpot Results** Code node -> **Filter Text for OpenAI** Code node.
   - Add **OpenAI Analysis** (`@n8n/n8n-nodes-langchain.openAi`) and connect it to **Decode AI Analysis** Code node.

3. **Configure SLA, Team Allocation, and Jira Operations:**
   - Connect **Decode AI Analysis** to **Compute SLA and Team Allocation** Code node -> **Generate Jira Payload** Code node -> **Search Jira for Duplicates** HTTP Request node -> **Parse Deduplication Result** Code node -> **Identify Duplicate Issues** IF node.
   - *Duplicate Branch (True):* Connect to **Build Duplicated Result** Code node -> **Notify Slack of Duplicate** Slack node.
   - *New Ticket Branch (False):* Connect to **Create Jira Issue4** HTTP Request node -> **Merge Jira Create Result4** Code node -> **Prepare SLA Stamp3** Code node -> **Should Stamp SLA?3** IF node.
   - *SLA Stamp True:* Connect to **Stamp SLA Custom Fields4** HTTP Request node -> **Restore After SLA Stamp3** Code node -> **Prepare Attachments4** Code node.
   - *SLA Stamp False:* Connect directly to **Prepare Attachments4** Code node -> **Skip Attachments?4** IF node.
   - *Attachments False:* Connect to **Normalize Before Slack3** Code node.
   - *Attachments True:* Connect to **Upload Jira Attachment4** HTTP Request node -> **Restore After Upload3** Code node -> **Normalize Before Slack3** Code node.

4. **Configure Notifications and Form Responses:**
   - From **Normalize Before Slack3**, branch to:
     - **Slack Ticket Notification4** Slack node -> **Merge Ticket with Slack** Merge node (Input 1).
     - **Build Primary Result** Code node -> **Merge Ticket with Slack** Merge node (Input 0).
   - Connect **Notify Slack of Duplicate** and **Merge Ticket with Slack** into **Merge All Results** Code node.
   - Connect **Merge All Results** to **Is Form Channel?** IF node.
   - *Form Channel True:* Connect to **Reply Form Success** Form node (`webhookId: form-success-ending`).

5. **Configure Periodic SLA Breach Monitoring:**
   - Add **Every 10 Minutes1** Schedule Trigger -> **Load SLA Config1** Code node -> **Search Breached Tickets1** HTTP Request node -> **Split Breached Issues1** Code node.
   - From **Split Breached Issues1**, branch to:
     - **Slack SLA Alert1** Slack node.
     - **Has Manager?1** IF node -> **Notify Manager1** Slack node.
     - **Raise Priority?1** IF node:
       - *True:* Connect to **Raise Jira Priority1** HTTP Request node -> **Restore Before Mark1** Code node -> **Mark SLA Notified1** HTTP Request node.
       - *False:* Connect to **Pass Ticket (No Raise)** Code node -> **Mark SLA Notified1** HTTP Request node.
   - Connect **Mark SLA Notified1** to **Summarize Mark Result** Code node.

6. **Configure Global Error Handling:**
   - Add **Error Trigger** -> **Format Error Payload** Code node -> **Slack Error Channel** Slack node (`webhookId: 451718ba-c741-41ba-bbd6-aad0a4e0019d`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| WAHA API Authentication | Requires selecting the 'WAHA Header Auth' credential (Header Auth, Name=`X-Api-Key`, Value=plain WAHA key) on the `Fetch WAHA Media` node after import. |
| Jira and Slack Setup | Ensure Jira project keys, custom field IDs for SLA/AI/dedupe/channel, and Slack channel names (`#support-triage`, `#support-sla`, `#support-ops-errors`) are correctly mapped in the configuration nodes. |