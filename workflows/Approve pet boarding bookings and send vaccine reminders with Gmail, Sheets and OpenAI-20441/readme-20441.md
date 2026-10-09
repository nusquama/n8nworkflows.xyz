Approve pet boarding bookings and send vaccine reminders with Gmail, Sheets and OpenAI

https://n8nworkflows.xyz/workflows/approve-pet-boarding-bookings-and-send-vaccine-reminders-with-gmail--sheets-and-openai-20441


# Approve pet boarding bookings and send vaccine reminders with Gmail, Sheets and OpenAI

### 1. Workflow Overview

This workflow automates pet boarding booking requests by combining form intake, deterministic validation, AI-generated communications, and Google Sheets logging. It verifies pet vaccination records (rabies, DHPP, and bordetella) against requested stay dates, classifies bookings as approved, conditional, or blocked, drafts tailored owner replies using an AI model, logs all submissions, alerts kennel staff to non-compliant requests, and runs scheduled daily vaccine expiration warnings and weekly operational digests.

The workflow logic is categorized into three functional blocks:
- **1.1 Booking Intake and Validation:** Receives submissions via an online form, normalizes parameters, generates tracking metadata, and validates mandatory contact and date fields.
- **1.2 Vaccine Compliance and AI Processing:** Performs deterministic date calculations to check vaccine validity, uses an AI agent to draft personalized email replies, logs structured data to Google Sheets, dispatches emails to owners, and flags issues to the kennel.
- **1.3 Automated Expiry Monitoring and Reporting:** Manages scheduled operations including a daily watch to warn owners of upcoming vaccine expirations and a Monday morning operational digest for the kennel team.

---

### 2. Block-by-Block Analysis

#### 2.1 Booking Intake and Validation
**Overview:** Captures incoming boarding requests through a hosted form, structures input parameters, generates unique tracking identifiers, and checks for missing critical information.

**Nodes Involved:**
- `Sticky Note`
- `Boarding Request Form`
- `Normalize Booking`
- `Validate Booking`
- `Email Intake Problem to Kennel`

**Node Details:**

- **Sticky Note**
  - **Type and Role:** `n8n-nodes-base.stickyNote` | Documentation and setup guide for the entire workflow.
  - **Configuration:** Contains markdown text explaining workflow architecture, setup requirements, and customization options.
  - **Connections:** None (Visual reference).
  - **Edge Cases:** None.

- **Boarding Request Form**
  - **Type and Role:** `n8n-nodes-base.formTrigger` | Webhook-based entry point providing a public HTML form for pet owners.
  - **Configuration:** Configured with 10 form fields covering owner details, pet attributes, stay dates, vaccine expiry dates, and notes. Required fields include owner name, email, pet name, check-in date, and check-out date.
  - **Expressions/Variables:** Uses built-in n8n form parameters.
  - **Input/Output Connections:** Output connects to `Normalize Booking`.
  - **Edge Cases:** Unformatted date strings or dropped fields if form is submitted with empty required fields.

- **Normalize Booking**
  - **Type and Role:** `n8n-nodes-base.set` | Data transformer node that standardizes input keys and appends metadata.
  - **Configuration:** Maps payload values using fallback expressions to ensure consistent keys regardless of input source casing. Appends `received_at` timestamps, generated `booking_id` strings (`BKG-YYYYMMDD-###`), and a default `kennel_email` address.
  - **Expressions/Variables:** 
    - `={{ $json.owner_name || $json['Owner name'] || '' }}`
    - `={{ $now.toISO() }}`
    - `={{ 'BKG-' + $now.toFormat('yyyyMMdd') + '-' + (100 + Math.floor(Math.random() * 900)) }}`
    - `={{ 'user@example.com' }}`
  - **Input/Output Connections:** Input from `Boarding Request Form`; output connects to `Validate Booking`.
  - **Edge Cases:** Random number generator collision (mitigated by broad range).

- **Validate Booking**
  - **Type and Role:** `n8n-nodes-base.if` | Conditional branch router verifying data completeness.
  - **Configuration:** Evaluates whether `owner_name`, `owner_email`, `pet_name`, `check_in_date`, and `check_out_date` are non-empty.
  - **Expressions/Variables:** Checks node data parameters for string presence (`notEmpty` operation).
  - **Input/Output Connections:** Input from `Normalize Booking`. True branch routes to `Check Vaccine Validity`. False branch routes to `Email Intake Problem to Kennel`.
  - **Edge Cases:** Whitespace-only strings passing empty evaluations if not trimmed at the source.

- **Email Intake Problem to Kennel**
  - **Type and Role:** `n8n-nodes-base.gmail` | Notification dispatcher for incomplete submissions.
  - **Configuration:** Sends a plain text email to the kennel address detailing which required parameters were omitted.
  - **Credentials:** Uses Gmail OAuth2 credentials (`Gmail Fresh Sep05`).
  - **Expressions/Variables:** Retrieves `kennel_email` dynamically via `{{ $('Normalize Booking').first().json.kennel_email || 'user@example.com' }}`. Constructs a multi-line notification string using array joins and newline character codes.
  - **Input/Output Connections:** Input from the false branch of `Validate Booking`. Terminal node.
  - **Edge Cases:** OAuth token expiration or invalid recipient address.

---

#### 2.2 Vaccine Compliance and AI Processing
**Overview:** Analyzes vaccine expiry dates against stay schedules, uses an AI agent to draft customized email responses, logs transactions in Google Sheets, notifies pet owners, and alerts kennel staff to non-compliant requests.

**Nodes Involved:**
- `Check Vaccine Validity`
- `Draft Booking Decision Email`
- `Booking Draft Model`
- `Prepare Decision Email`
- `Log Booking to Register`
- `Email Decision to Owner`
- `Blocked Booking?`
- `Alert Kennel to Blocked Booking`

**Node Details:**

- **Check Vaccine Validity**
  - **Type and Role:** `n8n-nodes-base.code` | JavaScript execution node performing deterministic vaccine compliance rules.
  - **Configuration:** Parses rabies, DHPP, and bordetella expiry dates. Compares them against the check-out date. Classifies compliance as `compliant`, `expiring_soon`, or `noncompliant`, and outputs decisions as `approved`, `conditional`, or `blocked`.
  - **Expressions/Variables:** Processes JSON inputs via embedded JavaScript date arithmetic.
  - **Input/Output Connections:** Input from `Validate Booking` (True branch); output connects to `Draft Booking Decision Email`.
  - **Edge Cases:** Malformed date strings causing parsing failures default to missing record assumptions.

- **Draft Booking Decision Email**
  - **Type and Role:** `@n8n/n8n-nodes-langchain.agent` | LangChain AI agent node generating structured customer communication based on compliance outcomes.
  - **Configuration:** Instructs the chat model to write a strict JSON response containing a subject (max 10 words) and plain-text body (under 140 words) without markdown or em dashes.
  - **Expressions/Variables:** Injects summary text combining owner name, pet details, stay dates, decision status, and vaccine issues.
  - **Input/Output Connections:** Input from `Check Vaccine Validity`; connected to `Booking Draft Model` for language model services; output connects to `Prepare Decision Email`.
  - **Edge Cases:** Model returning invalid JSON or conversational filler instead of exact requested keys.

- **Booking Draft Model**
  - **Type and Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM integration providing the chat model engine for the AI agent.
  - **Configuration:** Configured with model identifier `gpt-6-luna`.
  - **Credentials:** Uses OpenAI API credentials (`jonathan`).
  - **Input/Output Connections:** Connected as an AI language model provider to `Draft Booking Decision Email`.
  - **Edge Cases:** API rate limits, token budget exhaustion, or upstream service outages.

- **Prepare Decision Email**
  - **Type and Role:** `n8n-nodes-base.code` | JavaScript node parsing AI agent JSON output and merging it with booking metadata.
  - **Configuration:** Safely parses the raw string output from the AI agent, falling back to default subject lines if parsing fails.
  - **Expressions/Variables:** Iterates through input items, targets `.output`, and extracts `subject` and `body`.
  - **Input/Output Connections:** Input from `Draft Booking Decision Email`; output connects to `Log Booking to Register`.
  - **Edge Cases:** Malformed JSON blocks returned by the LLM requiring catch-block fallbacks.

- **Log Booking to Register**
  - **Type and Role:** `n8n-nodes-base.googleSheets` | Data storage node appending complete booking payloads to a tracking spreadsheet.
  - **Configuration:** Targets spreadsheet `Pet Boarding Compliance 1003`, sheet name `Bookings`. Maps 19 specific column fields including booking identifiers, dates, statuses, notes, and drafted emails.
  - **Credentials:** Uses Google Sheets OAuth2 credentials (`Feedback Analyzer Sheets`).
  - **Expressions/Variables:** Maps node parameters directly to corresponding JSON keys from upstream data.
  - **Input/Output Connections:** Input from `Prepare Decision Email`; output connects to `Email Decision to Owner`.
  - **Edge Cases:** Schema mismatch, permission revocation on Google Sheets API, or sheet renaming errors.

- **Email Decision to Owner**
  - **Type and Role:** `n8n-nodes-base.gmail` | Email communication node sending the AI-drafted reply to the pet owner.
  - **Configuration:** Sets recipient, subject, and body using values prepared upstream. Sends plain text emails.
  - **Credentials:** Uses Gmail OAuth2 credentials (`Gmail Fresh Sep05`).
  - **Expressions/Variables:** 
    - `={{ $('Prepare Decision Email').first().json.owner_email }}`
    - `={{ $('Prepare Decision Email').first().json.email_subject }}`
    - `={{ $('Prepare Decision Email').first().json.email_body }}`
  - **Input/Output Connections:** Input from `Log Booking to Register`; output connects to `Blocked Booking?`.
  - **Edge Cases:** Invalid email addresses or mail server rejections.

- **Blocked Booking?**
  - **Type and Role:** `n8n-nodes-base.if` | Conditional router checking if a booking requires internal follow-up.
  - **Configuration:** Evaluates whether the decision from `Check Vaccine Validity` is unequal to `approved`.
  - **Expressions/Variables:** `={{ $('Check Vaccine Validity').first().json.decision }}` compared against `approved`.
  - **Input/Output Connections:** Input from `Email Decision to Owner`. True branch routes to `Alert Kennel to Blocked Booking`. False branch terminates.
  - **Edge Cases:** Strict string comparison matching case sensitivity issues.

- **Alert Kennel to Blocked Booking**
  - **Type and Role:** `n8n-nodes-base.gmail` | Internal notification node alerting kennel staff to non-approved bookings.
  - **Configuration:** Sends a summary email to the kennel address containing pet details, decision status, and vaccine issues.
  - **Credentials:** Uses Gmail OAuth2 credentials (`Gmail Fresh Sep05`).
  - **Expressions/Variables:** Dynamically constructs notification strings combining owner and pet identifiers.
  - **Input/Output Connections:** Input from the True branch of `Blocked Booking?`. Terminal node.
  - **Edge Cases:** Missing kennel email variable falling back to default placeholder addresses.

---

#### 2.3 Automated Expiry Monitoring and Reporting
**Overview:** Operates scheduled tasks to check for expiring vaccines daily and compile a weekly operational digest for kennel management every Monday.

**Nodes Involved:**
- `Daily Expiry Watch`
- `Read Booking Register`
- `Find Expiring Vaccines`
- `Any Expiring?`
- `Email Expiry Heads Up`
- `Weekly Kennel Digest`
- `Read Register for Digest`
- `Summarize Bookings`
- `Draft Kennel Digest`
- `Digest Model`
- `Prepare Digest Email`
- `Email Weekly Digest to Kennel`

**Node Details:**

- **Daily Expiry Watch**
  - **Type and Role:** `n8n-nodes-base.scheduleTrigger` | Cron-based trigger initiating daily execution cycles.
  - **Configuration:** Cron expression set to run every day at 07:00 (`0 7 * * *`).
  - **Input/Output Connections:** Output connects to `Read Booking Register`.
  - **Edge Cases:** Timezone mismatches between the server environment and local kennel hours.

- **Read Booking Register**
  - **Type and Role:** `n8n-nodes-base.googleSheets` | Data retrieval node fetching all logged rows from the bookings ledger.
  - **Configuration:** Reads data from spreadsheet `Pet Boarding Compliance 1003`, sheet `Bookings`.
  - **Credentials:** Uses Google Sheets OAuth2 credentials (`Feedback Analyzer Sheets`).
  - **Input/Output Connections:** Input from `Daily Expiry Watch`; output connects to `Find Expiring Vaccines`.
  - **Edge Cases:** Large data sets exceeding execution time limits or empty sheet responses.

- **Find Expiring Vaccines**
  - **Type and Role:** `n8n-nodes-base.code` | JavaScript processing node analyzing row data for upcoming vaccine expirations.
  - **Configuration:** Deduplicates records by owner email and pet name, retaining the newest submission. Calculates days until expiry for rabies, DHPP, and bordetella records, flagging items expiring within 30 days.
  - **Expressions/Variables:** Processes array objects via custom JavaScript logic.
  - **Input/Output Connections:** Input from `Read Booking Register`; output connects to `Any Expiring?`.
  - **Edge Cases:** Incorrect date formatting in historical spreadsheet rows causing `NaN` calculations.

- **Any Expiring?**
  - **Type and Role:** `n8n-nodes-base.if` | Conditional branch validating whether any notification items were identified.
  - **Configuration:** Checks if `reminder_subject` is non-empty.
  - **Expressions/Variables:** `={{ $json.reminder_subject }}` evaluated for emptiness.
  - **Input/Output Connections:** Input from `Find Expiring Vaccines`. True branch routes to `Email Expiry Heads Up`. False branch terminates.
  - **Edge Cases:** Empty datasets bypassing notification steps correctly.

- **Email Expiry Heads Up**
  - **Type and Role:** `n8n-nodes-base.gmail` | Email communication node sending vaccine renewal reminders to pet owners.
  - **Configuration:** Sends plain text messages using dynamically generated subjects and message bodies.
  - **Credentials:** Uses Gmail OAuth2 credentials (`Gmail Fresh Sep05`).
  - **Expressions/Variables:** 
    - `={{ $json.owner_email }}`
    - `={{ $json.reminder_subject }}`
    - `={{ $json.reminder_body }}`
  - **Input/Output Connections:** Input from the True branch of `Any Expiring?`. Terminal node.
  - **Edge Cases:** Invalid email destinations causing bounce errors.

- **Weekly Kennel Digest**
  - **Type and Role:** `n8n-nodes-base.scheduleTrigger` | Cron-based trigger initiating weekly reporting cycles.
  - **Configuration:** Cron expression set to run every Monday at 09:00 (`0 9 * * 1`).
  - **Input/Output Connections:** Output connects to `Read Register for Digest`.
  - **Edge Cases:** Trigger timing dependent on workflow activation status.

- **Read Register for Digest**
  - **Type and Role:** `n8n-nodes-base.googleSheets` | Data retrieval node fetching the complete bookings ledger for weekly aggregation.
  - **Configuration:** Reads data from spreadsheet `Pet Boarding Compliance 1003`, sheet `Bookings`.
  - **Credentials:** Uses Google Sheets OAuth2 credentials (`Feedback Analyzer Sheets`).
  - **Input/Output Connections:** Input from `Weekly Kennel Digest`; output connects to `Summarize Bookings`.
  - **Edge Cases:** Empty spreadsheet tabs returning zero-length arrays.

- **Summarize Bookings**
  - **Type and Role:** `n8n-nodes-base.code` | JavaScript data aggregation node calculating weekly operational metrics.
  - **Configuration:** Aggregates booking totals, counts approvals, conditionals, and blocks, tallies pets with vaccines expiring within 30 days, and compiles an attention list.
  - **Expressions/Variables:** JavaScript aggregation code generating statistical summary objects.
  - **Input/Output Connections:** Input from `Read Register for Digest`; output connects to `Draft Kennel Digest`.
  - **Edge Cases:** Missing column values resulting in undefined metrics.

- **Draft Kennel Digest**
  - **Type and Role:** `@n8n/n8n-nodes-langchain.agent` | LangChain AI agent node generating operational summary emails for management.
  - **Configuration:** Instructs the chat model to write a strict JSON response containing a subject (max 12 words) and plain-text body (under 160 words) summarizing statistics.
  - **Expressions/Variables:** Injects summary statistics including week dates, total bookings, compliance counts, and pending attention items.
  - **Input/Output Connections:** Input from `Summarize Bookings`; connected to `Digest Model` for language model services; output connects to `Prepare Digest Email`.
  - **Edge Cases:** LLM output format degradation.

- **Digest Model**
  - **Type and Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM integration providing chat model support for weekly digest generation.
  - **Configuration:** Configured with model identifier `gpt-6-luna`.
  - **Credentials:** Uses OpenAI API credentials (`jonathan`).
  - **Input/Output Connections:** Connected as an AI language model provider to `Draft Kennel Digest`.
  - **Edge Cases:** API authentication or quota limitations.

- **Prepare Digest Email**
  - **Type and Role:** `n8n-nodes-base.code` | JavaScript utility node parsing AI agent output for the weekly digest.
  - **Configuration:** Safely parses JSON output strings and structures them into email parameters.
  - **Expressions/Variables:** Iterates items, parses JSON payloads, and assigns fallback titles.
  - **Input/Output Connections:** Input from `Draft Kennel Digest`; output connects to `Email Weekly Digest to Kennel`.
  - **Edge Cases:** Malformed JSON strings triggering safe fallback paths.

- **Email Weekly Digest to Kennel**
  - **Type and Role:** `n8n-nodes-base.gmail` | Email communication node delivering the operational digest to kennel management.
  - **Configuration:** Sends plain text emails using dynamic subject lines and message bodies.
  - **Credentials:** Uses Gmail OAuth2 credentials (`Gmail Fresh Sep05`).
  - **Expressions/Variables:** 
    - `={{ $('Summarize Bookings').first().json.kennel_email || 'user@example.com' }}`
    - `={{ $('Prepare Digest Email').first().json.email_subject }}`
    - `={{ $('Prepare Digest Email').first().json.email_body }}`
  - **Input/Output Connections:** Input from `Prepare Digest Email`. Terminal node.
  - **Edge Cases:** Invalid recipient configurations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation and setup guide for the entire workflow. | None | None | ## check pet vaccination records and approve boarding bookings with AI<br><br>### How it works<br><br>1. An owner submits a boarding booking on the hosted form with pet details, stay dates and vaccine expiry dates. The entry is validated and the kennel is emailed if details are missing.<br><br>2. A deterministic check compares rabies, DHPP and bordetella expiry dates against the stay. Each booking becomes approved, conditional (a vaccine expires around the stay) or blocked (a record is missing or expired).<br><br>3. An AI assistant writes the owner reply for the outcome: a confirmation, a nudge to renew one vaccine, or a request to send an updated record.<br><br>4. Every booking is logged to Google Sheets. Conditional or blocked bookings also alert the kennel inbox so staff can chase the record.<br><br>5. A daily 07:00 watch emails owners whose pet has a vaccine expiring within 30 days. On Monday at 09:00 an AI digest summarizes the week.<br><br>### Setup steps<br><br>- Create the Google Sheet (tab Bookings) with the columns used in the Google Sheets nodes and paste the spreadsheet id into those nodes.<br><br>- Add Google Sheets, Gmail and an [OI] compatible chat model credential (gpt-4o-mini or similar) and select them in the nodes.<br><br>- Set the kennel inbox in the Normalize Booking node (kennel_email) so internal alerts reach the right mailbox.<br><br>- Publish the form and share its link, then activate the workflow.<br><br>### Customization<br><br>- Change the required vaccines or the 14 day borderline window in the Check Vaccine Validity node.<br><br>- Change the 30 day heads up window or the digest schedule in the schedule triggers. |
| Boarding Request Form | n8n-nodes-base.formTrigger | Webhook-based entry point providing a public HTML form for pet owners. | None | Normalize Booking | |
| Normalize Booking | n8n-nodes-base.set | Data transformer node that standardizes input keys and appends metadata. | Boarding Request Form | Validate Booking | |
| Validate Booking | n8n-nodes-base.if | Conditional branch router verifying data completeness. | Normalize Booking | Check Vaccine Validity,<br>Email Intake Problem to Kennel | |
| Email Intake Problem to Kennel | n8n-nodes-base.gmail | Notification dispatcher for incomplete submissions. | Validate Booking (False) | None | |
| Check Vaccine Validity | n8n-nodes-base.code | JavaScript execution node performing deterministic vaccine compliance rules. | Validate Booking (True) | Draft Booking Decision Email | |
| Draft Booking Decision Email | @n8n/n8n-nodes-langchain.agent | LangChain AI agent node generating structured customer communication based on compliance outcomes. | Check Vaccine Validity | Prepare Decision Email | |
| Booking Draft Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM integration providing the chat model engine for the AI agent. | None | Draft Booking Decision Email | |
| Prepare Decision Email | n8n-nodes-base.code | JavaScript node parsing AI agent JSON output and merging it with booking metadata. | Draft Booking Decision Email | Log Booking to Register | |
| Log Booking to Register | n8n-nodes-base.googleSheets | Data storage node appending complete booking payloads to a tracking spreadsheet. | Prepare Decision Email | Email Decision to Owner | |
| Email Decision to Owner | n8n-nodes-base.gmail | Email communication node sending the AI-drafted reply to the pet owner. | Log Booking to Register | Blocked Booking? | |
| Blocked Booking? | n8n-nodes-base.if | Conditional router checking if a booking requires internal follow-up. | Email Decision to Owner | Alert Kennel to Blocked Booking | |
| Alert Kennel to Blocked Booking | n8n-nodes-base.gmail | Internal notification node alerting kennel staff to non-approved bookings. | Blocked Booking? (True) | None | |
| Daily Expiry Watch | n8n-nodes-base.scheduleTrigger | Cron-based trigger initiating daily execution cycles. | None | Read Booking Register | |
| Read Booking Register | n8n-nodes-base.googleSheets | Data retrieval node fetching all logged rows from the bookings ledger. | Daily Expiry Watch | Find Expiring Vaccines | |
| Find Expiring Vaccines | n8n-nodes-base.code | JavaScript processing node analyzing row data for upcoming vaccine expirations. | Read Booking Register | Any Expiring? | |
| Any Expiring? | n8n-nodes-base.if | Conditional branch validating whether any notification items were identified. | Find Expiring Vaccines | Email Expiry Heads Up | |
| Email Expiry Heads Up | n8n-nodes-base.gmail | Email communication node sending vaccine renewal reminders to pet owners. | Any Expiring? (True) | None | |
| Weekly Kennel Digest | n8n-nodes-base.scheduleTrigger | Cron-based trigger initiating weekly reporting cycles. | None | Read Register for Digest | |
| Read Register for Digest | n8n-nodes-base.googleSheets | Data retrieval node fetching the complete bookings ledger for weekly aggregation. | Weekly Kennel Digest | Summarize Bookings | |
| Summarize Bookings | n8n-nodes-base.code | JavaScript data aggregation node calculating weekly operational metrics. | Read Register for Digest | Draft Kennel Digest | |
| Draft Kennel Digest | @n8n/n8n-nodes-langchain.agent | LangChain AI agent node generating operational summary emails for management. | Summarize Bookings | Prepare Digest Email | |
| Digest Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM integration providing chat model support for weekly digest generation. | None | Draft Kennel Digest | |
| Prepare Digest Email | n8n-nodes-base.code | JavaScript utility node parsing AI agent output for the weekly digest. | Draft Kennel Digest | Email Weekly Digest to Kennel | |
| Email Weekly Digest to Kennel | n8n-nodes-base.gmail | Email communication node delivering the operational digest to kennel management. | Prepare Digest Email | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the entire workflow manually in n8n:

1. **Create Infrastructure:** Set up a Google Sheets document with a tab named `Bookings`. Add columns: `booking_id`, `received_at`, `owner_name`, `owner_email`, `pet_name`, `species`, `check_in_date`, `check_out_date`, `rabies_expiry`, `dhpp_expiry`, `bordetella_expiry`, `compliance`, `decision`, `issues`, `notes`, `kennel_email`, `email_subject`, `email_body`, and `last_watch_at`.
2. **Configure Credentials:** Set up valid credentials for Google Sheets OAuth2, Gmail OAuth2, and OpenAI API within your n8n instance.
3. **Build Branch 1: Form Intake and Validation**
   - **Node 1 (`Boarding Request Form`):** Create a Form Trigger node. Configure form fields for owner name, owner email, pet name, species, check-in date, check-out date, rabies expiry, DHPP expiry, bordetella expiry, and notes.
   - **Node 2 (`Normalize Booking`):** Add a Set node to normalize parameters and append fields: `received_at` (`$now.toISO()`), `booking_id` (`BKG-...`), and `kennel_email` (`user@example.com`). Connect `Boarding Request Form` output to this node.
   - **Node 3 (`Validate Booking`):** Add an IF node to check that `owner_name`, `owner_email`, `pet_name`, `check_in_date`, and `check_out_date` are not empty. Connect `Normalize Booking` output to this node.
   - **Node 4 (`Email Intake Problem to Kennel`):** Add a Gmail node configured to send a text email to the kennel address if validation fails. Connect the `false` output of `Validate Booking` to this node.
4. **Build Branch 2: Compliance and AI Processing**
   - **Node 5 (`Check Vaccine Validity`):** Add a Code node running JavaScript that compares rabies, DHPP, and bordetella expiry dates against the check-out date, assigning compliance and decision values. Connect the `true` output of `Validate Booking` to this node.
   - **Node 6 (`Booking Draft Model`):** Add an OpenAI Chat Model node using model `gpt-6-luna` and your OpenAI credentials.
   - **Node 7 (`Draft Booking Decision Email`):** Add an Advanced AI Agent node. Set prompt type to define, configure system instructions to output strictly JSON with keys `subject` and `body`, and link `Booking Draft Model` to its AI language model input. Connect `Check Vaccine Validity` output to this node.
   - **Node 8 (`Prepare Decision Email`):** Add a Code node to parse the JSON output from the AI agent and merge it with base record values. Connect `Draft Booking Decision Email` output to this node.
   - **Node 9 (`Log Booking to Register`):** Add a Google Sheets node set to `Append` operation. Select your spreadsheet and `Bookings` tab. Map all columns from input data. Connect `Prepare Decision Email` output to this node.
   - **Node 10 (`Email Decision to Owner`):** Add a Gmail node to send the drafted decision email to the pet owner (`owner_email`). Connect `Log Booking to Register` output to this node.
   - **Node 11 (`Blocked Booking?`):** Add an IF node checking if `decision` does not equal `approved`. Connect `Email Decision to Owner` output to this node.
   - **Node 12 (`Alert Kennel to Blocked Booking`):** Add a Gmail node to send an action alert to the kennel address. Connect the `true` output of `Blocked Booking?` to this node.
5. **Build Branch 3: Daily Expiry Watch**
   - **Node 13 (`Daily Expiry Watch`):** Add a Schedule Trigger node set to cron expression `0 7 * * *`.
   - **Node 14 (`Read Booking Register`):** Add a Google Sheets node set to read all rows from the `Bookings` tab. Connect `Daily Expiry Watch` output to this node.
   - **Node 15 (`Find Expiring Vaccines`):** Add a Code node to deduplicate rows by owner and pet, and identify vaccines expiring within 30 days. Connect `Read Booking Register` output to this node.
   - **Node 16 (`Any Expiring?`):** Add an IF node checking if `reminder_subject` is not empty. Connect `Find Expiring Vaccines` output to this node.
   - **Node 17 (`Email Expiring Heads Up`):** Add a Gmail node to email owners about expiring vaccines. Connect the `true` output of `Any Expiring?` to this node.
6. **Build Branch 4: Weekly Kennel Digest**
   - **Node 18 (`Weekly Kennel Digest`):** Add a Schedule Trigger node set to cron expression `0 9 * * 1`.
   - **Node 19 (`Read Register for Digest`):** Add a Google Sheets node set to read rows from the `Bookings` tab. Connect `Weekly Kennel Digest` output to this node.
   - **Node 20 (`Summarize Bookings`):** Add a Code node to aggregate weekly booking statistics and pending action items. Connect `Read Register for Digest` output to this node.
   - **Node 21 (`Digest Model`):** Add an OpenAI Chat Model node using model `gpt-6-luna`.
   - **Node 22 (`Draft Kennel Digest`):** Add an Advanced AI Agent node configured to generate a JSON weekly summary (`subject` and `body`). Link `Digest Model` to its AI language model input. Connect `Summarize Bookings` output to this node.
   - **Node 23 (`Prepare Digest Email`):** Add a Code node to parse the AI digest output. Connect `Draft Kennel Digest` output to this node.
   - **Node 24 (`Email Weekly Digest to Kennel`):** Add a Gmail node to send the weekly digest email to the kennel management address. Connect `Prepare Digest Email` output to this node.
7. **Finalize Setup:** Review all node connections, ensure timezones match your operating region, and activate the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n workflow automation platform. | [n8n Platform Link](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Professional consultation and business assessments. | [Consultation Booking Page](https://khmuhtadin.com/consultation/) |