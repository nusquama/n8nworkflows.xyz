Manage lead follow-ups with Google Sheets, SMTP, and IMAP email

https://n8nworkflows.xyz/workflows/manage-lead-follow-ups-with-google-sheets--smtp--and-imap-email-19680


# Manage lead follow-ups with Google Sheets, SMTP, and IMAP email

### 1. Workflow Overview

This workflow automates the entire lead intake, acknowledgment, and follow-up lifecycle for a professional service practice (such as a law firm). Its primary purpose is to eliminate manual tracking overhead, ensure consistent and compliant communication cadence, and automatically halt outreach the moment a lead responds. 

The logic is partitioned into four distinct functional blocks:

- **1.1 Lead Form Submission & Instant Acknowledgment:** Captures incoming lead submissions via webhook, standardizes lead fields, formats practice-area-specific acknowledgment emails, passes them through an external compliance guardrail sub-workflow, sends approved emails via SMTP, and logs the lead in Google Sheets.
- **1.2 Daily Follow-Up Sequence Check:** Runs on a daily schedule to query the tracking sheet, evaluates unresponsive leads against configured time-offset stages, builds tailored follow-up messages, runs compliance checks, sends approved follow-ups, and updates tracking records.
- **1.3 Final Stage No-Response Alert:** Evaluates whether a lead has reached the final outreach stage without replying, and if so, dispatches an internal alert email to the attorney.
- **1.4 Inbound Email Reply Processing:** Continuously monitors the sending mailbox via IMAP for incoming replies, parses the sender's address, cross-references it against tracked leads, and updates the Google Sheets tracker to flag the lead as "responded," terminating active follow-up sequences.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Form Submission & Instant Acknowledgment
- **Overview:** Receives raw lead submissions, formats structured lead objects, generates compliant acknowledgment emails based on practice areas, verifies opt-out compliance via sub-workflow, delivers the message, and registers the lead in a central Google Sheets tracker.
- **Nodes Involved:** 
  - `When Lead Form Is Submitted`
  - `Parse Incoming Lead Data`
  - `Build Acknowledgment Email Content`
  - `Check Ack Email Compliance`
  - `If Ack Email Is Approved`
  - `Send Acknowledgment Email`
  - `Skip Ack Email For Opt-Out Lead`
  - `Append Lead To Follow-Up Sheet`
- **Node Details:**
  - **When Lead Form Is Submitted**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration Choices:* Listens for incoming HTTP `POST` requests on path `lead-follow-up-intake`.
    - *Key Expressions/Variables:* None.
    - *Connections:* Input: None; Output: `Parse Incoming Lead Data`.
    - *Edge Cases/Failures:* Invalid payload format or network timeouts from form providers.
  - **Parse Incoming Lead Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Extracts and sanitizes request body fields (name, email, phone, practice area) and generates a unique `lead_id` and ISO timestamp.
    - *Key Expressions/Variables:* `$json.body || $json`, `Date.now()`.
    - *Connections:* Input: `When Lead Form Is Submitted`; Output: `Build Acknowledgment Email Content`.
    - *Edge Cases/Failures:* Missing expected properties in payload resulting in empty strings.
  - **Build Acknowledgment Email Content**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Templating Engine)
    - *Configuration Choices:* Maps practice areas (e.g., family law, estate planning) to specific templated response texts and wraps them in a responsive HTML structure.
    - *Key Expressions/Variables:* `$vars.FIRM_NAME`, `$json.practice_area`.
    - *Connections:* Input: `Parse Incoming Lead Data`; Output: `Check Ack Email Compliance`.
    - *Edge Cases/Failures:* Unhandled practice area falling back to default message templates.
  - **Check Ack Email Compliance**
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow Invocation)
    - *Configuration Choices:* Calls an external compliance/opt-out guardrail workflow dynamically by ID.
    - *Key Expressions/Variables:* `={{ $vars.GUARDRAIL_WORKFLOW_ID }}`, passing `channel`, `message`, `recipient`, and `template_id`.
    - *Connections:* Input: `Build Acknowledgment Email Content`; Output: `If Ack Email Is Approved`.
    - *Sub-workflow Reference:* Invokes compliance verification sub-workflow; expects an `approved` boolean output.
    - *Edge Cases/Failures:* Sub-workflow unavailability, missing workflow ID variable.
  - **If Ack Email Is Approved**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates whether the guardrail approval flag evaluates to true.
    - *Key Expressions/Variables:* `={{ $json.approved }}`.
    - *Connections:* Input: `Check Ack Email Compliance`; Outputs: `Send Acknowledgment Email` (True), `Skip Ack Email For Opt-Out Lead` (False).
    - *Edge Cases/Failures:* Type mismatch on the boolean evaluation.
  - **Send Acknowledgment Email**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (SMTP Output)
    - *Configuration Choices:* Sends HTML/text formatted email to the lead using institutional SMTP credentials.
    - *Key Expressions/Variables:* `{{ $('Build Acknowledgment Email Content').item.json.email_html }}`, `{{ $vars.FIRM_FROM_EMAIL }}`.
    - *Connections:* Input: `If Ack Email Is Approved` (True); Output: `Append Lead To Follow-Up Sheet`.
    - *Edge Cases/Failures:* SMTP authentication errors, invalid sender/recipient addresses, rate limits.
  - **Skip Ack Email For Opt-Out Lead**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Pass-through / Flow Control)
    - *Configuration Choices:* Acts as an endpoint for rejected emails, bypassing sending while allowing sheet logging to proceed.
    - *Connections:* Input: `If Ack Email Is Approved` (False); Output: `Append Lead To Follow-Up Sheet`.
  - **Append Lead To Follow-Up Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Database / Storage)
    - *Configuration Choices:* Appends a new tracking row to the "Lead Follow-Up Tracker" sheet with initial state (`responded: false`).
    - *Key Expressions/Variables:* `={{ $vars.LEAD_FOLLOWUP_SHEET_ID }}`, referencing upstream parsed lead data.
    - *Connections:* Input: `Send Acknowledgment Email` or `Skip Ack Email For Opt-Out Lead`; Output: None.
    - *Edge Cases/Failures:* Google API quota limits, invalid spreadsheet ID, sheet permission revocation.

#### 2.2 Daily Follow-Up Sequence Check
- **Overview:** Periodically evaluates all recorded leads, calculates elapsed time, determines eligibility for sequential follow-up stages, verifies compliance, sends emails via SMTP, and updates the tracking database.
- **Nodes Involved:**
  - `Trigger Daily Follow-Up Check`
  - `Fetch Follow-Up Tracker Data`
  - `Calculate Due Follow-Up Stage`
  - `Build Follow-Up Email Content`
  - `Check Follow-Up Email Compliance`
  - `If Follow-Up Email Approved`
  - `Send Follow-Up Email`
  - `Skip Follow-Up For Opted-Out Lead`
  - `Update Tracker Post Follow-Up`
- **Node Details:**
  - **Trigger Daily Follow-Up Check**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Timer Trigger)
    - *Configuration Choices:* Configured via cron expression to execute daily at 09:00 AM (`0 9 * * *`).
    - *Connections:* Input: None; Output: `Fetch Follow-Up Tracker Data`.
  - **Fetch Follow-Up Tracker Data**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration Choices:* Reads all rows from the "Lead Follow-Up Tracker" sheet.
    - *Key Expressions/Variables:* `={{ $vars.LEAD_FOLLOWUP_SHEET_ID }}`.
    - *Connections:* Input: `Trigger Daily Follow-Up Check`; Output: `Calculate Due Follow-Up Stage`.
  - **Calculate Due Follow-Up Stage**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Business Logic Engine)
    - *Configuration Choices:* Parses schedule offset configurations (`FOLLOW_UP_STAGES`), filters out responded leads, and computes whether leads are due for subsequent outreach stages.
    - *Key Expressions/Variables:* `$vars.FOLLOW_UP_STAGES`.
    - *Connections:* Input: `Fetch Follow-Up Tracker Data`; Outputs: `Build Follow-Up Email Content`, `If Lead Reached Final Stage`.
    - *Edge Cases/Failures:* Malformed cron offset strings or corrupt date strings in tracking records.
  - **Build Follow-Up Email Content**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Templating Engine)
    - *Configuration Choices:* Selects the appropriate message body corresponding to the specific stage (`due_stage_label`) and structures it into an HTML template.
    - *Key Expressions/Variables:* `$json.due_stage_label`, `$vars.FIRM_NAME`.
    - *Connections:* Input: `Calculate Due Follow-Up Stage`; Output: `Check Follow-Up Email Compliance`.
  - **Check Follow-Up Email Compliance**
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow Invocation)
    - *Configuration Choices:* Dispatches follow-up content to the centralized compliance guardrail sub-workflow.
    - *Key Expressions/Variables:* `={{ $vars.GUARDRAIL_WORKFLOW_ID }}`.
    - *Connections:* Input: `Build Follow-Up Email Content`; Output: `If Follow-Up Email Approved`.
  - **If Follow-Up Email Approved**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates the approval boolean returned by the compliance check.
    - *Connections:* Input: `Check Follow-Up Email Compliance`; Outputs: `Send Follow-Up Email` (True), `Skip Follow-Up For Opted-Out Lead` (False).
  - **Send Follow-Up Email**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (SMTP Output)
    - *Configuration Choices:* Delivers the approved follow-up email to the lead.
    - *Connections:* Input: `If Follow-Up Email Approved` (True); Output: `Update Tracker Post Follow-Up`.
  - **Skip Follow-Up For Opted-Out Lead**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Pass-through)
    - *Configuration Choices:* Null operation node terminating unapproved follow-up pathways.
    - *Connections:* Input: `If Follow-Up Email Approved` (False); Output: None.
  - **Update Tracker Post Follow-Up**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Database / Storage)
    - *Configuration Choices:* Performs an upsert operation using `lead_id` to update `last_sent_at` and `last_stage_sent`.
    - *Key Expressions/Variables:* `={{ $('Build Follow-Up Email Content').item.json.lead_id }}`, `={{ new Date().toISOString() }}`.
    - *Connections:* Input: `Send Follow-Up Email`; Output: None.

#### 2.3 Final Stage No-Response Alert
- **Overview:** Checks if a lead has reached the terminal stage of the follow-up cadence without responding, and triggers an internal notification email to the attorney.
- **Nodes Involved:**
  - `If Lead Reached Final Stage`
  - `Build No-Response Alert Email`
  - `Send No-Response Alert Email`
  - `Skip Alert For Non-Final Stage`
- **Node Details:**
  - **If Lead Reached Final Stage**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates the boolean flag `is_final_stage` passed from the stage calculation node.
    - *Key Expressions/Variables:* `={{ $json.is_final_stage }}`.
    - *Connections:* Input: `Calculate Due Follow-Up Stage`; Outputs: `Build No-Response Alert Email` (True), `Skip Alert For Non-Final Stage` (False).
  - **Build No-Response Alert Email**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Templating Engine)
    - *Configuration Choices:* Constructs an internal notification payload summarizing lead contact details and unresponsiveness.
    - *Key Expressions/Variables:* `$vars.FIRM_ATTORNEY_EMAIL || $vars.FIRM_EMAIL`.
    - *Connections:* Input: `If Lead Reached Final Stage` (True); Output: `Send No-Response Alert Email`.
  - **Send No-Response Alert Email**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (SMTP Output)
    - *Configuration Choices:* Sends the alert email to internal stakeholders via SMTP.
    - *Connections:* Input: `Build No-Response Alert Email`; Output: None.
  - **Skip Alert For Non-Final Stage**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Pass-through)
    - *Configuration Choices:* Terminates execution pathways for non-terminal stages.
    - *Connections:* Input: `If Lead Reached Final Stage` (False); Output: None.

#### 2.4 Inbound Email Reply Processing
- **Overview:** Continuously polls the designated inbox via IMAP, extracts sender addresses from incoming replies, matches them against tracked leads, and updates the tracker to halt further automated outreach.
- **Nodes Involved:**
  - `When Lead Replies By Email`
  - `Parse Reply Email Sender`
  - `Read Tracker For Reply Match`
  - `Match Reply To Lead Tracker`
  - `Mark Lead As Responded In Sheet`
- **Node Details:**
  - **When Lead Replies By Email**
    - *Type and Technical Role:* `n8n-nodes-base.emailReadImap` (IMAP Trigger)
    - *Configuration Choices:* Monitors the `INBOX` folder with a reconnection interval of 60 seconds and marks processed messages as read.
    - *Connections:* Input: None; Output: `Parse Reply Email Sender`.
    - *Edge Cases/Failures:* IMAP server connectivity loss, incorrect authentication credentials, folder path mismatches.
  - **Parse Reply Email Sender**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Extraction)
    - *Configuration Choices:* Extracts the plain text or structured email address from the incoming message header.
    - *Key Expressions/Variables:* `$json.from.value[0].address`.
    - *Connections:* Input: `When Lead Replies By Email`; Output: `Read Tracker For Reply Match`.
  - **Read Tracker For Reply Match**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration Choices:* Reads all rows from the tracking sheet to obtain active lead email records.
    - *Key Expressions/Variables:* `={{ $vars.LEAD_FOLLOWUP_SHEET_ID }}`.
    - *Connections:* Input: `Parse Reply Email Sender`; Output: `Match Reply To Lead Tracker`.
  - **Match Reply To Lead Tracker**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Matching Logic)
    - *Configuration Choices:* Compares incoming sender email against tracked lead records, returning matched row data or an empty array if unmatched.
    - *Connections:* Input: `Read Tracker For Reply Match`; Output: `Mark Lead As Responded In Sheet`.
    - *Edge Cases/Failures:* Unmatched replies return zero items, gracefully terminating the branch without errors.
  - **Mark Lead As Responded In Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Database Update)
    - *Configuration Choices:* Updates the tracking record matching `lead_id`, setting `responded` to `true`.
    - *Key Expressions/Variables:* `={{ $json.lead_id }}`, `responded: true`.
    - *Connections:* Input: `Match Reply To Lead Tracker`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Setup Overview | None | None | ## Automated Lead Follow-Up Sequence<br><br>### How it works<br><br>1. The workflow triggers when a new lead form is submitted and processes the lead information to send an immediate acknowledgment email if compliance is met.<br>2. It logs lead details for follow-up tracking in a Google Sheets document.<br>3. A scheduled trigger runs daily to check which leads require follow-up emails based on the tracking data and sends compliant follow-up emails accordingly.<br>4. It monitors if the follow-up sequence reaches a final stage without a response and sends an alert email.<br>5. The workflow checks incoming email replies from leads, matches them to the tracked leads, and marks them as responded in the tracker to prevent further follow-ups.<br><br>### Setup steps<br><br>- [ ] Configure webhook credentials or endpoint URL for the 'When Lead Form Submitted' trigger node.<br>- [ ] Connect and authorize access to the Google Sheets containing lead tracking information for nodes interacting with sheets.<br>- [ ] Set up and authorize email sending service for 'Send Instant Acknowledgment Email', 'Send Follow-Up Email', and 'Send No-Response Alert Email' nodes.<br>- [ ] Configure email reading via IMAP in 'When Lead Replies Via Email' node with appropriate credentials and folder settings.<br>- [ ] Ensure the compliance sub-workflows invoked by 'Check Compliance Before Ack Email' and 'Check Compliance Before Follow-Up Email' are accessible and properly configured.<br><br>### Customization<br><br>Modify custom code nodes to adjust email content, follow-up timing, or compliance rules to fit specific business needs. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section Header: Intake & Ack | None | None | ## Lead form submission and acknowledgment<br><br>Handles lead form data reception, parsing, building, compliance checking, and sending of the instant acknowledgment email, then logs the lead for tracking. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section Header: Daily Follow-Up | None | None | ## Daily follow-up sequence check<br><br>Scheduled trigger fetches lead tracking data, computes which leads are due for follow-up, then builds and verifies follow-up emails before sending or skipping based on compliance. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section Header: Final Alert | None | None | ## Final stage no-response alert<br><br>Checks if leads have reached the final follow-up stage without response and builds and sends alert emails accordingly. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section Header: Reply Processing | None | None | ## Lead reply processing<br><br>Handles incoming email replies from leads, parsing sender information, matching replies to tracked leads, and marking the leads as responded in the tracker to stop follow-ups. |
| When Lead Form Is Submitted | n8n-nodes-base.webhook | Webhook Intake | None | Parse Incoming Lead Data | Wire your inquiry form (JotForm, Typeform, Fluent Forms, or similar) to POST here with the lead's name, email, phone, and practice area. |
| Parse Incoming Lead Data | n8n-nodes-base.code | Data Standardization | When Lead Form Is Submitted | Build Acknowledgment Email Content | Field names here match common inquiry-form defaults (JotForm/Typeform/Fluent Forms) -- adjust to your actual form's field names if they differ. |
| Build Acknowledgment Email Content | n8n-nodes-base.code | Email Templating | Parse Incoming Lead Data | Check Ack Email Compliance | One templated body per practice area (family law, estate planning) plus a general default -- edit BODY_BY_AREA here for different copy. |
| Check Ack Email Compliance | n8n-nodes-base.executeWorkflow | Compliance Verification | Build Acknowledgment Email Content | If Ack Email Is Approved | Real marketing email to a real lead -- opt-out and disclaimer handling here are genuinely load-bearing. |
| If Ack Email Is Approved | n8n-nodes-base.if | Conditional Routing | Check Ack Email Compliance | Send Acknowledgment Email,<br>Skip Ack Email For Opt-Out Lead | Mirrors the guardrail's approved/opted-out result. |
| Send Acknowledgment Email | n8n-nodes-base.emailSend | SMTP Email Dispatch | If Ack Email Is Approved | Append Lead To Follow-Up Sheet | Reaches back to 'Build Instant Acknowledgment Email' by name, not $json, since the guardrail call in between replaces $json with its own response. |
| Skip Ack Email For Opt-Out Lead | n8n-nodes-base.noOp | Opt-out Bypass | If Ack Email Is Approved | Append Lead To Follow-Up Sheet | Rare for a first-time inquiry, but respected the same as any other opt-out check in this catalog. |
| Append Lead To Follow-Up Sheet | n8n-nodes-base.googleSheets | Lead Logging | Send Acknowledgment Email,<br>Skip Ack Email For Opt-Out Lead | None | Logs every lead regardless of the acknowledgment email's approval outcome -- follow-up tracking shouldn't depend on that. Reads from 'Parse Lead Submission' by name since the guardrail call upstream replaced $json. |
| Trigger Daily Follow-Up Check | n8n-nodes-base.scheduleTrigger | Daily Cron Trigger | None | Fetch Follow-Up Tracker Data | Runs daily, 9am by default. |
| Fetch Follow-Up Tracker Data | n8n-nodes-base.googleSheets | Batch Tracker Read | Trigger Daily Follow-Up Check | Calculate Due Follow-Up Stage | Reads every tracked lead -- filtered to who's actually due for a follow-up in the next step. |
| Calculate Due Follow-Up Stage | n8n-nodes-base.code | Stage Calculation Logic | Fetch Follow-Up Tracker Data | Build Follow-Up Email Content,<br>If Lead Reached Final Stage | FOLLOW_UP_STAGES controls the whole sequence — number of stages, spacing, and labels. A responded lead is always skipped regardless of timing. |
| Build Follow-Up Email Content | n8n-nodes-base.code | Follow-Up Templating | Calculate Due Follow-Up Stage | Check Follow-Up Email Compliance | One templated body per stage -- edit BODY_BY_STAGE here for different copy per touch. |
| Check Follow-Up Email Compliance | n8n-nodes-base.executeWorkflow | Compliance Verification | Build Follow-Up Email Content | If Follow-Up Email Approved | Real marketing email to a real lead -- opt-out and disclaimer handling here are genuinely load-bearing. |
| If Follow-Up Email Approved | n8n-nodes-base.if | Conditional Routing | Check Follow-Up Email Compliance | Send Follow-Up Email,<br>Skip Follow-Up For Opted-Out Lead | Mirrors the guardrail's approved/opted-out result. |
| Send Follow-Up Email | n8n-nodes-base.emailSend | SMTP Email Dispatch | If Follow-Up Email Approved | Update Tracker Post Follow-Up | Reaches back to 'Build Follow-Up Email' by name, not $json, since the guardrail call in between replaces $json with its own response. |
| Skip Follow-Up For Opted-Out Lead | n8n-nodes-base.noOp | Opt-out Bypass | If Follow-Up Email Approved | None | Doesn't advance the tracker's stage -- mark the lead responded=true directly in the sheet to stop future attempts cleanly. |
| Update Tracker Post Follow-Up | n8n-nodes-base.googleSheets | Tracker Update | Send Follow-Up Email | None | Only updates last_stage_sent and last_sent_at, matched on lead_id -- never touches responded or the other columns. |
| If Lead Reached Final Stage | n8n-nodes-base.if | Terminal Stage Branching | Calculate Due Follow-Up Stage | Build No-Response Alert Email,<br>Skip Alert For Non-Final Stage | Only the last configured stage (day7 by default) triggers the attorney no-response alert -- earlier stages just send their own follow-up, no alert. |
| Build No-Response Alert Email | n8n-nodes-base.code | Alert Templating | If Lead Reached Final Stage | Send No-Response Alert Email | Internal alert to the attorney -- no guardrail wrap needed, this doesn't go to the lead. |
| Send No-Response Alert Email | n8n-nodes-base.emailSend | SMTP Alert Dispatch | Build No-Response Alert Email | None | Internal notification only, not sent to the lead -- no guardrail wrap needed. |
| Skip Alert For Non-Final Stage | n8n-nodes-base.noOp | Pass-through | If Lead Reached Final Stage | None | An earlier stage (day1/day3) -- its own follow-up email already covers this, no attorney alert needed yet. |
| When Lead Replies By Email | n8n-nodes-base.emailReadImap | IMAP Inbox Reader | None | Parse Reply Email Sender | Watches the same inbox FIRM_FROM_EMAIL sends from, via IMAP. Credential: Credentials → New → IMAP → your mailbox's server/login details. |
| Parse Reply Email Sender | n8n-nodes-base.code | Sender Extraction | When Lead Replies By Email | Read Tracker For Reply Match | Extracts the replying address so it can be matched against the tracker in the next step. |
| Read Tracker For Reply Match | n8n-nodes-base.googleSheets | Batch Tracker Read | Parse Reply Email Sender | Match Reply To Lead Tracker | Reads every tracked lead so the replying address can be matched by email. |
| Match Reply To Lead Tracker | n8n-nodes-base.code | Email Matching Logic | Read Tracker For Reply Match | Mark Lead As Responded In Sheet | Matches by exact email address — a reply from an address not in the tracker is simply ignored. |
| Mark Lead As Responded In Sheet | n8n-nodes-base.googleSheets | Tracker Status Update | Match Reply To Lead Tracker | None | Only updates the responded flag, matched on lead_id -- stops all future follow-up emails to this lead immediately. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Workflow & Variables
1. Create a new workflow in n8n.
2. Define required workflow environment variables in your n8n instance or configuration:
   - `FIRM_NAME`: Name of your practice or firm (e.g., "Acme Law").
   - `FIRM_FROM_EMAIL`: Outbound sender email address.
   - `FIRM_ATTORNEY_EMAIL`: Internal attorney notification address.
   - `LEAD_FOLLOWUP_SHEET_ID`: Target Google Sheet document ID.
   - `GUARDRAIL_WORKFLOW_ID`: ID of your compliance/opt-out sub-workflow.
   - `FOLLOW_UP_STAGES`: Timing configuration string (default: `1|day1;3|day3;7|day7`).

#### Step 2: Build the Intake and Acknowledgment Branch
1. **Webhook Node (`When Lead Form Is Submitted`)**: Create a Webhook node, set HTTP method to `POST`, and set Path to `lead-follow-up-intake`.
2. **Code Node (`Parse Incoming Lead Data`)**: Add a Code node connected downstream. Insert JavaScript logic to extract name, email, phone, and practice area, assigning a unique `lead_id` and timestamp.
3. **Code Node (`Build Acknowledgment Email Content`)**: Add a Code node to generate subject lines, raw message text, and responsive HTML templates based on practice area.
4. **Execute Workflow Node (`Check Ack Email Compliance`)**: Add an Execute Workflow node configured to evaluate compliance via expression using `{{ $vars.GUARDRAIL_WORKFLOW_ID }}` and pass mapping parameters (`channel`, `message`, `recipient`, `template_id`).
5. **If Node (`If Ack Email Is Approved`)**: Add an If node evaluating `{{ $json.approved }}` as true.
6. **Email Send Node (`Send Acknowledgment Email`)**: Add an Email Send (SMTP) node connected to the *True* branch. Configure SMTP credentials, pulling HTML, text, subject, and recipient values from the preceding builder node using item references (e.g., `{{ $('Build Acknowledgment Email Content').item.json... }}`).
7. **No-Op Node (`Skip Ack Email For Opt-Out Lead`)**: Add a No-Op node connected to the *False* branch of the approval If node.
8. **Google Sheets Node (`Append Lead To Follow-Up Sheet`)**: Add a Google Sheets node configured for the `append` operation, connecting both the email sending node and the No-Op node to its input. Map columns (`lead_id`, `name`, `email`, `phone`, `practice_area`, `submitted_at`, `responded` [default: false], `last_sent_at`, `last_stage_sent`) referencing the parsed lead data.

#### Step 3: Build the Daily Follow-Up Sequence Check Branch
1. **Schedule Trigger Node (`Trigger Daily Follow-Up Check`)**: Add a Schedule Trigger node, setting cron expression to `0 9 * * *`.
2. **Google Sheets Node (`Fetch Follow-Up Tracker Data`)**: Add a Google Sheets node configured to `read` all rows from the "Lead Follow-Up Tracker" sheet using `{{ $vars.LEAD_FOLLOWUP_SHEET_ID }}`.
3. **Code Node (`Calculate Due Follow-Up Stage`)**: Add a Code node to parse `FOLLOW_UP_STAGES`, filter out responded leads, compute days elapsed since submission, and determine the next due outreach stage. Connect the output to both the follow-up email builder and the final stage condition.
4. **Code Node (`Build Follow-Up Email Content`)**: Add a Code node to build stage-specific follow-up email subjects, body text, and HTML layouts.
5. **Execute Workflow Node (`Check Follow-Up Email Compliance`)**: Add an Execute Workflow node referencing `GUARDRAIL_WORKFLOW_ID` to verify follow-up compliance.
6. **If Node (`If Follow-Up Email Approved`)**: Add an If node checking `{{ $json.approved }}`.
7. **Email Send Node (`Send Follow-Up Email`)**: Add an SMTP email node connected to the *True* branch to dispatch the follow-up message.
8. **No-Op Node (`Skip Follow-Up For Opted-Out Lead`)**: Add a No-Op node connected to the *False* branch.
9. **Google Sheets Node (`Update Tracker Post Follow-Up`)**: Add a Google Sheets node configured for `appendOrUpdate` (matching on `lead_id`) to record `last_sent_at` and `last_stage_sent`. Connect this to the follow-up email send node.

#### Step 4: Build the Final Stage No-Response Alert Branch
1. **If Node (`If Lead Reached Final Stage`)**: Add an If node connected to the output of `Calculate Due Follow-Up Stage`, evaluating `={{ $json.is_final_stage }}`.
2. **Code Node (`Build No-Response Alert Email`)**: Add a Code node to construct internal alert messages and HTML templates directed at the attorney.
3. **Email Send Node (`Send No-Response Alert Email`)**: Add an SMTP email node connected to the *True* branch, addressing the internal recipient (`$vars.FIRM_ATTORNEY_EMAIL`).
4. **No-Op Node (`Skip Alert For Non-Final Stage`)**: Add a No-Op node connected to the *False* branch of the final stage condition.

#### Step 5: Build the Inbound Email Reply Processing Branch
1. **Email Read (IMAP) Node (`When Lead Replies By Email`)**: Add an IMAP trigger node configured with IMAP credentials, watching the `INBOX` folder, with post-process action set to read.
2. **Code Node (`Parse Reply Email Sender`)**: Add a Code node to extract the clean sender email address from the incoming message headers.
3. **Google Sheets Node (`Tracker For Reply Match`)**: Add a Google Sheets node configured to `read` all tracking rows from the spreadsheet.
4. **Code Node (`Match Reply To Lead Tracker`)**: Add a Code node to match the replying sender's email against the active lead tracking dataset.
5. **Google Sheets Node (`Mark Lead As Responded In Sheet`)**: Add a Google Sheets node configured for `appendOrUpdate` (matching on `lead_id`) to update the `responded` column to `true`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Compliance Guidelines | Ensure all automated email sequences adhere to relevant bar association ethics opinions (e.g., ABA Formal Opinion 512 regarding autonomous legal communication and opt-out mechanisms). |
| Google Sheets Schema | The target spreadsheet must contain a sheet named exactly `Lead Follow-Up Tracker` with columns: `lead_id`, `name`, `email`, `phone`, `practice_area`, `submitted_at`, `responded`, `last_sent_at`, and `last_stage_sent`. |