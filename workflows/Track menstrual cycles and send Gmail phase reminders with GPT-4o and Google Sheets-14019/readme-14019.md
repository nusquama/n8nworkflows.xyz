Track menstrual cycles and send Gmail phase reminders with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/track-menstrual-cycles-and-send-gmail-phase-reminders-with-gpt-4o-and-google-sheets-14019


# Track menstrual cycles and send Gmail phase reminders with GPT-4o and Google Sheets

# 1. Workflow Overview

This workflow manages a menstrual-cycle reminder system using an n8n form, Google Sheets, Gmail, and OpenAI GPT-4o. It collects subscriber details, calculates key cycle dates, stores subscriber records, sends an onboarding email, delivers phase-based reminder emails on the correct dates, and sends a personalized weekly wellness digest every Sunday.

Its main use cases are:
- Collecting menstrual cycle subscription data from users
- Tracking each subscriber’s cycle milestones
- Sending scheduled reminder emails for important cycle events
- Generating AI-personalized weekly wellness emails based on the current phase

## 1.1 Subscriber Intake and Onboarding
A form collects name, email, last period start date, and cycle length. The workflow calculates derived dates, saves the subscriber to Google Sheets, and sends a welcome email summarizing the cycle.

## 1.2 Daily Phase Reminder Automation
Every day at 8 AM, the workflow reads all subscribers, filters active ones, calculates whether today matches a reminder event, prevents duplicate sends within the same cycle, then sends the appropriate reminder email and logs it.

## 1.3 Weekly AI Wellness Digest
Every Sunday at 9 AM, the workflow reads all active subscribers, determines each subscriber’s current phase, creates a GPT-4o prompt, generates phase-aware wellness guidance, formats it into HTML, emails it, and logs the send.

## 1.4 Data Persistence and Logging
Google Sheets is used as both the subscriber database and the send log. The `Subscribers` sheet stores cycle data and last-send markers, while `Send Log` records every reminder and weekly digest sent.

---

# 2. Block-by-Block Analysis

## 2.1 Subscriber Intake and Onboarding

### Overview
This block is triggered by a public n8n form. It normalizes user input, calculates cycle milestones, stores the subscriber record in Google Sheets, and sends a welcome email containing the predicted cycle overview.

### Nodes Involved
- Cycle Wellness Form
- Calculate Cycle Dates
- Save to Subscribers
- Build Welcome Email
- Send Welcome Email

### Node Details

#### 2.1.1 Cycle Wellness Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`; entry-point node that exposes a hosted form and triggers the workflow on submission.
- **Configuration choices:**
  - Form title: `Period Health & Cycle Wellness`
  - Description: invites users to track their cycle and receive reminders
  - Fields:
    - `Full Name` (required)
    - `Email Address` as email type (required)
    - `Last Period Start Date` as date type (required)
    - `Cycle Length (days)` as number type (required)
- **Key expressions or variables used:** none in-node; downstream nodes reference submitted field labels exactly.
- **Input and output connections:**
  - No input; this is an entry point
  - Outputs to `Calculate Cycle Dates`
- **Version-specific requirements:** type version `2.3`
- **Edge cases or potential failure types:**
  - Field label changes will break downstream code if expressions still reference old labels
  - Invalid date format may still cause later code issues depending on browser/input handling
  - Public form endpoint may receive spam if exposed openly
- **Sub-workflow reference:** none

#### 2.1.2 Calculate Cycle Dates
- **Type and technical role:** `n8n-nodes-base.code`; computes derived cycle dates from form data.
- **Configuration choices:**
  - Reads first incoming item only
  - Extracts:
    - `Full Name`
    - `Email Address`
    - `Last Period Start Date`
    - `Cycle Length (days)`
  - Defaults cycle length to `28` if parsing fails
  - Computes:
    - `next_period`
    - `ovulation_date` = next period minus 14 days
    - `fertile_start` = ovulation minus 5 days
    - `fertile_end` = ovulation plus 1 day
    - `pms_start` = next period minus 5 days
    - `subscribed_date`
    - `last_email_sent` empty string
    - `active` = `'true'`
  - Formats dates as `YYYY-MM-DD`
- **Key expressions or variables used:**
  - `$input.first().json[...]`
  - `parseInt(...) || 28`
  - `toISOString().split('T')[0]`
- **Input and output connections:**
  - Input from `Cycle Wellness Form`
  - Output to `Save to Subscribers`
- **Version-specific requirements:** type version `2`
- **Edge cases or potential failure types:**
  - `new Date(lastPeriodRaw)` can produce invalid dates if source input is malformed
  - `toISOString()` throws on invalid Date objects
  - Timezone conversion can shift date output by one day in some environments because of ISO conversion
  - Cycle lengths outside realistic ranges are not validated
- **Sub-workflow reference:** none

#### 2.1.3 Save to Subscribers
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends subscriber records into the `Subscribers` sheet.
- **Configuration choices:**
  - Operation: `append`
  - Document: `Period Health Tracker`
  - Sheet: `Subscribers`
  - Explicit column mapping for:
    - `name`
    - `email`
    - `last_period_date`
    - `cycle_length`
    - `subscribed_date`
    - `next_period`
    - `ovulation_date`
    - `pms_start`
    - `fertile_start`
    - `fertile_end`
    - `last_email_sent`
    - `active`
- **Key expressions or variables used:**
  - `={{ $json.fieldName }}`
- **Input and output connections:**
  - Input from `Calculate Cycle Dates`
  - Output to `Build Welcome Email`
- **Version-specific requirements:** type version `4.7`
- **Edge cases or potential failure types:**
  - Google Sheets auth failure
  - Missing worksheet or changed column names
  - Duplicate subscriptions are not prevented
  - Concurrent writes may create duplicate rows
- **Sub-workflow reference:** none

#### 2.1.4 Build Welcome Email
- **Type and technical role:** `n8n-nodes-base.code`; generates HTML and subject line for the onboarding email.
- **Configuration choices:**
  - Builds branded HTML email with cards for:
    - next period
    - ovulation day
    - fertile window
    - PMS start
  - Adds unsubscribe-style footer text: “Reply STOP to unsubscribe.”
  - Subject: `🌸 Welcome {name} — Your Cycle Overview is Ready`
- **Key expressions or variables used:**
  - `const data = $input.first().json`
  - template literal interpolation of cycle date fields
- **Input and output connections:**
  - Input from `Save to Subscribers`
  - Output to `Send Welcome Email`
- **Version-specific requirements:** type version `2`
- **Edge cases or potential failure types:**
  - Missing `name` or cycle fields produce awkward but non-fatal output
  - User-provided values are inserted directly into HTML without escaping
- **Sub-workflow reference:** none

#### 2.1.5 Send Welcome Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the welcome HTML email through Gmail.
- **Configuration choices:**
  - Recipient: `{{$json.email}}`
  - Subject: `{{$json.subject}}`
  - Message body: `{{$json.htmlEmail}}`
  - Uses Gmail OAuth2 credentials
- **Key expressions or variables used:**
  - `={{ $json.email }}`
  - `={{ $json.subject }}`
  - `={{ $json.htmlEmail }}`
- **Input and output connections:**
  - Input from `Build Welcome Email`
  - No downstream node
- **Version-specific requirements:** type version `2.2`
- **Edge cases or potential failure types:**
  - Gmail OAuth token expiry or revoked permissions
  - Gmail send quotas or anti-spam restrictions
  - HTML may be rendered differently by recipients
- **Sub-workflow reference:** none

---

## 2.2 Daily Phase Reminder Automation

### Overview
This block runs every day at 8 AM. It reads subscribers from Google Sheets, filters to active users, calculates which reminder—if any—should be sent today, ensures the reminder has not already been sent for the current cycle, then sends and logs the email.

### Nodes Involved
- Daily 8AM Trigger
- Read All Subscribers
- Filter Active Subscribers
- Calculate Today's Phase
- IF Email to Send
- Build Reminder Email
- Send Reminder Email
- Update Last Email Sent
- Log to Send Log

### Node Details

#### 2.2.1 Daily 8AM Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; launches the daily reminder branch on a schedule.
- **Configuration choices:**
  - Runs every day at hour `8`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - No input; entry point
  - Output to `Read All Subscribers`
- **Version-specific requirements:** type version `1.3`
- **Edge cases or potential failure types:**
  - Runs according to instance timezone; incorrect timezone will shift send times
  - Missed executions if n8n is offline at trigger time
- **Sub-workflow reference:** none

#### 2.2.2 Read All Subscribers
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from the `Subscribers` sheet.
- **Configuration choices:**
  - Document: `Period Health Tracker`
  - Sheet: `Subscribers`
  - Default read behavior with no filtering in-node
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Daily 8AM Trigger`
  - Output to `Filter Active Subscribers`
- **Version-specific requirements:** type version `4.7`
- **Edge cases or potential failure types:**
  - Auth failure
  - Empty sheet
  - Changed column headers causing downstream logic to fail
- **Sub-workflow reference:** none

#### 2.2.3 Filter Active Subscribers
- **Type and technical role:** `n8n-nodes-base.filter`; keeps only active subscribers.
- **Configuration choices:**
  - Condition checks `{{$json.active}}` with boolean “true” operator
- **Key expressions or variables used:**
  - `={{ $json.active }}`
- **Input and output connections:**
  - Input from `Read All Subscribers`
  - Output to `Calculate Today's Phase`
- **Version-specific requirements:** type version `2.3`
- **Edge cases or potential failure types:**
  - The workflow stores `active` as string `'true'`, so strict boolean comparison may behave unexpectedly depending on n8n coercion behavior
  - If a row has blank or inconsistent values (`TRUE`, `True`, `1`), filtering may exclude valid subscribers
- **Sub-workflow reference:** none

#### 2.2.4 Calculate Today's Phase
- **Type and technical role:** `n8n-nodes-base.code`; determines whether each subscriber should receive a reminder today.
- **Configuration choices:**
  - Normalizes all dates to midnight
  - Computes `cycleDay` from `today - last_period_date + 1`
  - Computes `threeDaysBefore` next period
  - Assigns `emailType` if today matches:
    - `period_start`
    - `period_incoming`
    - `ovulation`
    - `pms_prep`
  - Assigns a current `phase` label:
    - `PMS`
    - `Menstrual`
    - `Fertile`
    - `Ovulation`
    - default `Follicular`
  - Skips subscribers with no email event today
  - Prevents duplicates if `last_email_sent` already contains `{emailType}_{next_period}`
  - If no subscribers qualify, returns one item with `{ skip: true }`
- **Key expressions or variables used:**
  - `$input.all()`
  - `u.last_email_sent`
  - `fmt(today) === fmt(nextPeriod)` etc.
- **Input and output connections:**
  - Input from `Filter Active Subscribers`
  - Output to `IF Email to Send`
- **Version-specific requirements:** type version `2`
- **Edge cases or potential failure types:**
  - Stored cycle dates never roll forward automatically after a period cycle completes; reminders depend on static saved dates
  - Phase logic order is slightly inconsistent: `Fertile` is checked before `Ovulation` in effect because ovulation day also falls in fertile range, so `phase` may remain `Fertile` on ovulation day
  - Invalid or empty date fields can break `toISOString()`
  - Negative cycle days are possible if dates are future-dated
- **Sub-workflow reference:** none

#### 2.2.5 IF Email to Send
- **Type and technical role:** `n8n-nodes-base.if`; removes placeholder skip items.
- **Configuration choices:**
  - Condition: `{{$json.skip}} != "true"`
- **Key expressions or variables used:**
  - `={{ $json.skip }}`
- **Input and output connections:**
  - Input from `Calculate Today's Phase`
  - True branch goes to `Build Reminder Email`
  - False branch unused
- **Version-specific requirements:** type version `2.3`
- **Edge cases or potential failure types:**
  - `skip` is returned as boolean-like data only in one synthetic item; condition compares string text, which works only if coercion matches expectations
- **Sub-workflow reference:** none

#### 2.2.6 Build Reminder Email
- **Type and technical role:** `n8n-nodes-base.code`; builds one of four HTML reminder templates.
- **Configuration choices:**
  - Uses `emailType` to choose among:
    - `period_incoming`
    - `period_start`
    - `ovulation`
    - `pms_prep`
  - Each template defines:
    - subject
    - heading
    - gradient colors
    - intro text
    - 5 practical tips
    - closing sentence
  - Appends cycle day and current phase in footer area of message body
- **Key expressions or variables used:**
  - `const template = emailTemplates[data.emailType]`
  - `data.nextPeriodStr`, `data.fertileStartStr`, `data.fertileEndStr`, `data.phase`, `data.cycleDay`
- **Input and output connections:**
  - Input from `IF Email to Send`
  - Output to `Send Reminder Email`
- **Version-specific requirements:** type version `2`
- **Edge cases or potential failure types:**
  - If `emailType` is unexpected, `template` becomes undefined and code crashes
  - User values are inserted into HTML without escaping
- **Sub-workflow reference:** none

#### 2.2.7 Send Reminder Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the phase reminder email.
- **Configuration choices:**
  - Recipient: `{{$json.email}}`
  - Subject: `{{$json.subject}}`
  - Message body: `{{$json.htmlEmail}}`
- **Key expressions or variables used:**
  - `={{ $json.email}}`
  - `={{ $json.subject }}`
  - `={{ $json.htmlEmail }}`
- **Input and output connections:**
  - Input from `Build Reminder Email`
  - Output to `Update Last Email Sent`
- **Version-specific requirements:** type version `2.2`
- **Edge cases or potential failure types:**
  - Gmail quota/auth errors
  - HTML rendering differences
  - If multiple items send in one execution, Gmail rate limits may appear
- **Sub-workflow reference:** none

#### 2.2.8 Update Last Email Sent
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates the subscriber record to prevent duplicate sends in the same cycle.
- **Configuration choices:**
  - Operation: `update`
  - Matches row by `email`
  - Updates `last_email_sent` to:
    - `{emailType}_{next_period}`
  - Sheet: `Subscribers`
- **Key expressions or variables used:**
  - `={{ $('Build Reminder Email').item.json.email }}`
  - `={{ $('Build Reminder Email').item.json.emailType + '_' + $('Build Reminder Email').item.json.next_period }}`
- **Input and output connections:**
  - Input from `Send Reminder Email`
  - Output to `Log to Send Log`
- **Version-specific requirements:** type version `4.7`
- **Edge cases or potential failure types:**
  - If duplicate rows share the same email, updates may affect ambiguous targets
  - If `email` column header changes, matching fails
  - If Gmail send succeeds but Sheets update fails, duplicates can be sent on later runs
- **Sub-workflow reference:** none

#### 2.2.9 Log to Send Log
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends a send record into the log sheet.
- **Configuration choices:**
  - Operation: `append`
  - Document: `Period Health Tracker`
  - Sheet: `Send Log`
  - Logs:
    - current date
    - name
    - email
    - phase
    - subject
    - cycle day
    - email type
- **Key expressions or variables used:**
  - `={{ new Date().toISOString().split('T')[0] }}`
  - Data references use `$('Build Reminder Email').item.json...`
- **Input and output connections:**
  - Input from `Update Last Email Sent`
  - No downstream node
- **Version-specific requirements:** type version `4.7`
- **Edge cases or potential failure types:**
  - Log append failure does not affect delivery already completed
  - ISO date may shift due to timezone
- **Sub-workflow reference:** none

---

## 2.3 Weekly AI Wellness Digest

### Overview
This block runs every Sunday at 9 AM. It calculates each active subscriber’s current phase, asks GPT-4o to generate a structured JSON wellness digest, converts that result into HTML, sends the email, and logs the send.

### Nodes Involved
- Weekly Sunday 9AM Trigger
- Read All Subscribers1
- Filter Active Subscribers1
- Calculate Phase & Build Prompt
- Wellness Coach
- Parse & Build Wellness Email
- Send Wellness Digest
- Log to Send Log1

### Node Details

#### 2.3.1 Weekly Sunday 9AM Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry point for the weekly digest branch.
- **Configuration choices:**
  - Interval field: `weeks`
  - Trigger hour: `9`
  - The node name indicates Sunday 9 AM, but the exact weekday behavior depends on how the schedule rule is interpreted by n8n and instance timezone
- **Key expressions or variables used:** none
- **Input and output connections:**
  - No input
  - Output to `Read All Subscribers1`
- **Version-specific requirements:** type version `1.3`
- **Edge cases or potential failure types:**
  - Timezone mismatch
  - If weekday is not explicitly configured, actual run timing should be validated in the editor
- **Sub-workflow reference:** none

#### 2.3.2 Read All Subscribers1
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads all subscriber rows.
- **Configuration choices:**
  - Same Google Sheet and `Subscribers` tab as the daily branch
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Weekly Sunday 9AM Trigger`
  - Output to `Filter Active Subscribers1`
- **Version-specific requirements:** type version `4.7`
- **Edge cases or potential failure types:** same as `Read All Subscribers`
- **Sub-workflow reference:** none

#### 2.3.3 Filter Active Subscribers1
- **Type and technical role:** `n8n-nodes-base.filter`; keeps active subscribers for weekly digests.
- **Configuration choices:**
  - Same active check as daily branch
- **Key expressions or variables used:**
  - `={{ $json.active }}`
- **Input and output connections:**
  - Input from `Read All Subscribers1`
  - Output to `Calculate Phase & Build Prompt`
- **Version-specific requirements:** type version `2.3`
- **Edge cases or potential failure types:**
  - Same string/boolean mismatch risk as in the daily branch
- **Sub-workflow reference:** none

#### 2.3.4 Calculate Phase & Build Prompt
- **Type and technical role:** `n8n-nodes-base.code`; calculates current phase and constructs the GPT prompt.
- **Configuration choices:**
  - Computes:
    - `phase`
    - `cycleDay`
    - `daysUntilPeriod`
  - Builds a long prompt instructing the model to act as a warm women’s health and wellness coach
  - Requires output strictly in JSON with keys:
    - `phase_summary`
    - `energy_tip`
    - `nutrition_tip`
    - `movement_tip`
    - `mindset_tip`
    - `weekly_affirmation`
    - `what_to_expect`
  - Includes style rules:
    - use her name
    - warm, sisterly, empowering tone
    - practical tips
    - each tip 1–2 sentences
- **Key expressions or variables used:**
  - `$input.all().map(...)`
  - `u.name`, `u.cycle_length`, computed phase metrics
- **Input and output connections:**
  - Input from `Filter Active Subscribers1`
  - Output to `Wellness Coach`
- **Version-specific requirements:** type version `2`
- **Edge cases or potential failure types:**
  - Static date model means phase accuracy degrades over time if subscriber dates are not refreshed each cycle
  - `phase` ordering may label ovulation day as `Fertile` before `Ovulation`
  - Negative `daysUntilPeriod` possible once stored dates are in the past
- **Sub-workflow reference:** none

#### 2.3.5 Wellness Coach
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; sends the prompt to OpenAI GPT-4o.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Response content is driven by `{{$json.prompt}}`
  - No built-in tools enabled
- **Key expressions or variables used:**
  - `={{ $json.prompt }}`
- **Input and output connections:**
  - Input from `Calculate Phase & Build Prompt`
  - Output to `Parse & Build Wellness Email`
- **Version-specific requirements:**
  - Requires LangChain OpenAI node package
  - Type version `2.1`
  - Requires valid OpenAI API credentials with model access to GPT-4o
- **Edge cases or potential failure types:**
  - OpenAI auth errors
  - Rate limits
  - Model may return non-JSON or fenced JSON despite prompt
  - Cost considerations if subscriber count is large
- **Sub-workflow reference:** none

#### 2.3.6 Parse & Build Wellness Email
- **Type and technical role:** `n8n-nodes-base.code`; parses the model output and renders a phase-themed HTML email.
- **Configuration choices:**
  - Reads model response from `items[0].json.output[0].content[0].text`
  - Strips Markdown code fences
  - Parses JSON
  - Pulls user metadata from `$('Calculate Phase & Build Prompt').item.json`
  - Selects colors and emoji by phase:
    - Menstrual
    - Follicular
    - Fertile
    - Ovulation
    - PMS
  - Builds branded digest email with:
    - phase summary
    - energy, nutrition, movement, mindset tips
    - expectation card
    - affirmation block
  - Subject line:
    - `{emoji} Your Weekly Wellness Digest — {phase} Phase · Day {cycleDay}`
- **Key expressions or variables used:**
  - `items[0].json.output[0].content[0].text`
  - `JSON.parse(cleaned)`
  - `$('Calculate Phase & Build Prompt').item.json`
- **Input and output connections:**
  - Input from `Wellness Coach`
  - Output to `Send Wellness Digest`
- **Version-specific requirements:** type version `2`
- **Edge cases or potential failure types:**
  - Very fragile parser path; any change in OpenAI node output structure breaks it
  - `JSON.parse` fails if model returns invalid JSON
  - If the workflow processes multiple subscribers in parallel, using `$('Calculate Phase & Build Prompt').item.json` must align correctly per item; mismatches can occur depending on execution context
- **Sub-workflow reference:** none

#### 2.3.7 Send Wellness Digest
- **Type and technical role:** `n8n-nodes-base.gmail`; emails the weekly digest.
- **Configuration choices:**
  - Recipient: `{{$json.email}}`
  - Subject: `{{$json.subject}}`
  - Message body: `{{$json.htmlEmail}}`
- **Key expressions or variables used:**
  - `={{ $json.email }}`
  - `={{ $json.subject }}`
  - `={{ $json.htmlEmail }}`
- **Input and output connections:**
  - Input from `Parse & Build Wellness Email`
  - Output to `Log to Send Log1`
- **Version-specific requirements:** type version `2.2`
- **Edge cases or potential failure types:** same Gmail risks as other send nodes
- **Sub-workflow reference:** none

#### 2.3.8 Log to Send Log1
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends weekly digest sends to the `Send Log` sheet.
- **Configuration choices:**
  - Operation: `append`
  - Logs current date, subscriber info, phase, subject, cycle day
  - Hardcodes `email_type` as `weekly_digest`
- **Key expressions or variables used:**
  - `={{ new Date().toISOString().split('T')[0] }}`
  - `={{ $('Parse & Build Wellness Email').item.json.name }}`
  - `=weekly_digest`
- **Input and output connections:**
  - Input from `Send Wellness Digest`
  - No downstream node
- **Version-specific requirements:** type version `4.7`
- **Edge cases or potential failure types:**
  - Logging can fail after email already sent
  - Date timezone issues similar to other logging nodes
- **Sub-workflow reference:** none

---

## 2.4 Documentation and In-Canvas Notes

### Overview
These nodes are non-executable sticky notes used to describe the overall design and the three main functional branches directly in the n8n canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### 2.4.1 Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation-only canvas annotation.
- **Configuration choices:**
  - Describes the overall weekly automation
  - Includes setup steps:
    1. Create Google Sheet with `Subscribers` and `Send Log`
    2. Connect Google Sheets, OpenAI, and Gmail credentials
    3. Test with own email
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none; visual only
- **Sub-workflow reference:** none

#### 2.4.2 Sticky Note1
- **Type and technical role:** sticky note; documents the intake/onboarding block.
- **Configuration choices:**
  - Explains collection of subscriber data
  - Explains calculation of cycle milestones
  - Explains save + welcome email flow
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 2.4.3 Sticky Note2
- **Type and technical role:** sticky note; documents the daily reminder block.
- **Configuration choices:**
  - Describes daily 8 AM execution
  - Lists supported email types
  - Notes duplicate send prevention
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### 2.4.4 Sticky Note3
- **Type and technical role:** sticky note; documents the weekly AI digest block.
- **Configuration choices:**
  - Describes Sunday 9 AM run
  - Notes GPT-4o-generated content
  - Notes adaptive email colors/design per phase
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Cycle Wellness Form | n8n-nodes-base.formTrigger | Public form entry point for new subscribers |  | Calculate Cycle Dates | Triggered when a subscriber submits the form. Collects name, email, last period date, and cycle length. Calculates next period, ovulation day, fertile window, and PMS start date automatically. Saves full profile to Subscribers sheet and sends a welcome email with their complete cycle overview. |
| Calculate Cycle Dates | n8n-nodes-base.code | Compute cycle milestone dates from form input | Cycle Wellness Form | Save to Subscribers | Triggered when a subscriber submits the form. Collects name, email, last period date, and cycle length. Calculates next period, ovulation day, fertile window, and PMS start date automatically. Saves full profile to Subscribers sheet and sends a welcome email with their complete cycle overview. |
| Save to Subscribers | n8n-nodes-base.googleSheets | Append subscriber profile to Google Sheets | Calculate Cycle Dates | Build Welcome Email | Triggered when a subscriber submits the form. Collects name, email, last period date, and cycle length. Calculates next period, ovulation day, fertile window, and PMS start date automatically. Saves full profile to Subscribers sheet and sends a welcome email with their complete cycle overview. |
| Build Welcome Email | n8n-nodes-base.code | Generate HTML onboarding email | Save to Subscribers | Send Welcome Email | Triggered when a subscriber submits the form. Collects name, email, last period date, and cycle length. Calculates next period, ovulation day, fertile window, and PMS start date automatically. Saves full profile to Subscribers sheet and sends a welcome email with their complete cycle overview. |
| Send Welcome Email | n8n-nodes-base.gmail | Send onboarding email through Gmail | Build Welcome Email |  | Triggered when a subscriber submits the form. Collects name, email, last period date, and cycle length. Calculates next period, ovulation day, fertile window, and PMS start date automatically. Saves full profile to Subscribers sheet and sends a welcome email with their complete cycle overview. |
| Daily 8AM Trigger | n8n-nodes-base.scheduleTrigger | Daily scheduler for reminder branch |  | Read All Subscribers | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Read All Subscribers | n8n-nodes-base.googleSheets | Load subscriber records for daily processing | Daily 8AM Trigger | Filter Active Subscribers | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Filter Active Subscribers | n8n-nodes-base.filter | Keep only active subscribers for daily reminders | Read All Subscribers | Calculate Today's Phase | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Calculate Today's Phase | n8n-nodes-base.code | Determine if a reminder should be sent today | Filter Active Subscribers | IF Email to Send | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| IF Email to Send | n8n-nodes-base.if | Remove placeholder skip items | Calculate Today's Phase | Build Reminder Email | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Build Reminder Email | n8n-nodes-base.code | Generate HTML for phase reminder emails | IF Email to Send | Send Reminder Email | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Send Reminder Email | n8n-nodes-base.gmail | Send reminder email | Build Reminder Email | Update Last Email Sent | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Update Last Email Sent | n8n-nodes-base.googleSheets | Update duplicate-send marker in subscriber sheet | Send Reminder Email | Log to Send Log | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Log to Send Log | n8n-nodes-base.googleSheets | Append reminder send event to log sheet | Update Last Email Sent |  | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Weekly Sunday 9AM Trigger | n8n-nodes-base.scheduleTrigger | Weekly scheduler for AI digest branch |  | Read All Subscribers1 | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |
| Read All Subscribers1 | n8n-nodes-base.googleSheets | Load subscriber records for weekly digest | Weekly Sunday 9AM Trigger | Filter Active Subscribers1 | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |
| Filter Active Subscribers1 | n8n-nodes-base.filter | Keep only active subscribers for weekly digests | Read All Subscribers1 | Calculate Phase & Build Prompt | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |
| Calculate Phase & Build Prompt | n8n-nodes-base.code | Compute current phase and create GPT prompt | Filter Active Subscribers1 | Wellness Coach | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |
| Wellness Coach | @n8n/n8n-nodes-langchain.openAi | Generate structured wellness guidance with GPT-4o | Calculate Phase & Build Prompt | Parse & Build Wellness Email | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |
| Parse & Build Wellness Email | n8n-nodes-base.code | Parse OpenAI output and render digest HTML | Wellness Coach | Send Wellness Digest | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |
| Send Wellness Digest | n8n-nodes-base.gmail | Send weekly AI-generated digest email | Parse & Build Wellness Email | Log to Send Log1 | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |
| Log to Send Log1 | n8n-nodes-base.googleSheets | Append weekly digest send event to log sheet | Send Wellness Digest |  | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for overall design and setup |  |  | ## Weekly Automation Every woman's cycle is different. This workflow tracks each subscriber's unique cycle, sends perfectly timed reminders before every phase, and delivers personalized weekly wellness tips powered by GPT-4o — all completely automatically. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for onboarding branch |  |  | Triggered when a subscriber submits the form. Collects name, email, last period date, and cycle length. Calculates next period, ovulation day, fertile window, and PMS start date automatically. Saves full profile to Subscribers sheet and sends a welcome email with their complete cycle overview. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for daily reminder branch |  |  | Runs every morning at 8AM. Reads all active subscribers, calculates today's cycle phase for each one, and sends the right email only on the right day — period incoming (3 days before), period start, ovulation alert, or PMS prep. Duplicate send prevention ensures each email type is only sent once per cycle. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for weekly AI digest branch |  |  | Runs every Sunday at 9AM. Calculates each subscriber's current cycle phase and generates a fully personalized wellness email using GPT-4o with energy, nutrition, movement, and mindset tips specific to their phase. Email design and colors adapt automatically to the current phase. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet**
   - Create one spreadsheet named something like `Period Health Tracker`.
   - Add two tabs:
     - `Subscribers`
     - `Send Log`

2. **Create the `Subscribers` sheet columns**
   - Add these headers in row 1:
     - `name`
     - `email`
     - `last_period_date`
     - `cycle_length`
     - `subscribed_date`
     - `next_period`
     - `ovulation_date`
     - `pms_start`
     - `fertile_start`
     - `fertile_end`
     - `last_email_sent`
     - `active`

3. **Create the `Send Log` sheet columns**
   - Add these headers:
     - `date`
     - `email`
     - `name`
     - `email_type`
     - `cycle_day`
     - `phase`
     - `subject`

4. **Connect credentials in n8n**
   - Add a **Google Sheets OAuth2** credential with access to the spreadsheet.
   - Add a **Gmail OAuth2** credential for sending emails.
   - Add an **OpenAI API** credential with access to `gpt-4o`.

5. **Create the form entry node**
   - Add a **Form Trigger** node.
   - Name it `Cycle Wellness Form`.
   - Set form title to `Period Health & Cycle Wellness`.
   - Set description to `Track your cycle and receive personalized health reminders and wellness tips.`
   - Add fields:
     1. Text field: `Full Name`, required
     2. Email field: `Email Address`, required
     3. Date field: `Last Period Start Date`, required
     4. Number field: `Cycle Length (days)`, required, placeholder `28`

6. **Create the cycle calculation node**
   - Add a **Code** node named `Calculate Cycle Dates`.
   - Connect `Cycle Wellness Form -> Calculate Cycle Dates`.
   - Paste logic that:
     - reads the submitted fields
     - parses cycle length with fallback `28`
     - computes `next_period`, `ovulation_date`, `fertile_start`, `fertile_end`, `pms_start`
     - stores `last_email_sent` as empty string
     - stores `active` as `'true'`
     - formats all dates as `YYYY-MM-DD`

7. **Create the subscriber save node**
   - Add a **Google Sheets** node named `Save to Subscribers`.
   - Connect `Calculate Cycle Dates -> Save to Subscribers`.
   - Set operation to `Append`.
   - Select the spreadsheet and `Subscribers` tab.
   - Map columns to the fields produced by the code node.

8. **Create the welcome email builder**
   - Add a **Code** node named `Build Welcome Email`.
   - Connect `Save to Subscribers -> Build Welcome Email`.
   - Configure it to create:
     - `htmlEmail`
     - `subject`
   - Include cards for next period, ovulation, fertile window, and PMS start.

9. **Create the welcome email sender**
   - Add a **Gmail** node named `Send Welcome Email`.
   - Connect `Build Welcome Email -> Send Welcome Email`.
   - Configure:
     - To: `{{$json.email}}`
     - Subject: `{{$json.subject}}`
     - Message: `{{$json.htmlEmail}}`
   - Attach Gmail OAuth2 credentials.

10. **Create the daily trigger**
    - Add a **Schedule Trigger** node named `Daily 8AM Trigger`.
    - Configure it to run every day at 8 AM.

11. **Create daily subscriber reader**
    - Add a **Google Sheets** node named `Read All Subscribers`.
    - Connect `Daily 8AM Trigger -> Read All Subscribers`.
    - Configure it to read from the `Subscribers` tab.

12. **Create the active subscriber filter for daily sends**
    - Add a **Filter** node named `Filter Active Subscribers`.
    - Connect `Read All Subscribers -> Filter Active Subscribers`.
    - Set condition to keep only rows where `active` is true.
    - Recommended improvement: compare as string `true` to match stored data exactly.

13. **Create the daily phase calculator**
    - Add a **Code** node named `Calculate Today's Phase`.
    - Connect `Filter Active Subscribers -> Calculate Today's Phase`.
    - Implement logic to:
      - normalize dates
      - compute `cycleDay`
      - calculate `threeDaysBefore`
      - assign `emailType` for:
        - period start
        - period incoming
        - ovulation
        - PMS prep
      - set current `phase`
      - skip if no email event today
      - skip if `last_email_sent` already equals or contains `{emailType}_{next_period}`
      - return `{ skip: true }` if nothing qualifies

14. **Create the send/no-send gate**
    - Add an **IF** node named `IF Email to Send`.
    - Connect `Calculate Today's Phase -> IF Email to Send`.
    - Configure the true path to continue when `skip` is not equal to `true`.

15. **Create the reminder email builder**
    - Add a **Code** node named `Build Reminder Email`.
    - Connect the true output of `IF Email to Send -> Build Reminder Email`.
    - Build an `emailTemplates` object keyed by:
      - `period_incoming`
      - `period_start`
      - `ovulation`
      - `pms_prep`
    - Each template should define subject, heading, colors, intro, tips, and closing.
    - Output `htmlEmail` and `subject`.

16. **Create the reminder sender**
    - Add a **Gmail** node named `Send Reminder Email`.
    - Connect `Build Reminder Email -> Send Reminder Email`.
    - Configure:
      - To: `{{$json.email}}`
      - Subject: `{{$json.subject}}`
      - Message: `{{$json.htmlEmail}}`

17. **Create the duplicate-send update node**
    - Add a **Google Sheets** node named `Update Last Email Sent`.
    - Connect `Send Reminder Email -> Update Last Email Sent`.
    - Set operation to `Update`.
    - Select `Subscribers`.
    - Match rows using the `email` column.
    - Update:
      - `email`
      - `last_email_sent = {{$json.emailType + '_' + $json.next_period}}`
    - If you use cross-node expressions instead of current-item fields, ensure item pairing is preserved.

18. **Create daily send logging**
    - Add a **Google Sheets** node named `Log to Send Log`.
    - Connect `Update Last Email Sent -> Log to Send Log`.
    - Set operation to `Append`.
    - Select `Send Log`.
    - Map:
      - `date` = today
      - `name`
      - `email`
      - `phase`
      - `subject`
      - `cycle_day`
      - `email_type`

19. **Create the weekly trigger**
    - Add a **Schedule Trigger** node named `Weekly Sunday 9AM Trigger`.
    - Configure it to run weekly at 9 AM on Sunday.
    - Verify the weekday explicitly in your n8n version.

20. **Create weekly subscriber reader**
    - Add a **Google Sheets** node named `Read All Subscribers1`.
    - Connect `Weekly Sunday 9AM Trigger -> Read All Subscribers1`.
    - Read from `Subscribers`.

21. **Create weekly active filter**
    - Add a **Filter** node named `Filter Active Subscribers1`.
    - Connect `Read All Subscribers1 -> Filter Active Subscribers1`.
    - Keep only active rows.

22. **Create phase calculator and prompt builder**
    - Add a **Code** node named `Calculate Phase & Build Prompt`.
    - Connect `Filter Active Subscribers1 -> Calculate Phase & Build Prompt`.
    - Compute:
      - current `phase`
      - `cycleDay`
      - `daysUntilPeriod`
    - Build a prompt requesting JSON-only output with:
      - `phase_summary`
      - `energy_tip`
      - `nutrition_tip`
      - `movement_tip`
      - `mindset_tip`
      - `weekly_affirmation`
      - `what_to_expect`

23. **Create the OpenAI generation node**
    - Add an **OpenAI** node from the LangChain set.
    - Name it `Wellness Coach`.
    - Connect `Calculate Phase & Build Prompt -> Wellness Coach`.
    - Choose model `gpt-4o`.
    - Send `{{$json.prompt}}` as the content.
    - Attach OpenAI credentials.

24. **Create the AI output parser and HTML builder**
    - Add a **Code** node named `Parse & Build Wellness Email`.
    - Connect `Wellness Coach -> Parse & Build Wellness Email`.
    - Parse the OpenAI response text.
    - Strip code fences like ```json if present.
    - `JSON.parse()` the result.
    - Build HTML using phase-specific colors and emoji.
    - Output:
      - `htmlEmail`
      - `subject`
      - subscriber metadata

25. **Create the weekly digest sender**
    - Add a **Gmail** node named `Send Wellness Digest`.
    - Connect `Parse & Build Wellness Email -> Send Wellness Digest`.
    - Configure:
      - To: `{{$json.email}}`
      - Subject: `{{$json.subject}}`
      - Message: `{{$json.htmlEmail}}`

26. **Create weekly log node**
    - Add a **Google Sheets** node named `Log to Send Log1`.
    - Connect `Send Wellness Digest -> Log to Send Log1`.
    - Append to `Send Log`.
    - Map:
      - `date`
      - `name`
      - `email`
      - `phase`
      - `subject`
      - `cycle_day`
      - `email_type = weekly_digest`

27. **Optional: add sticky notes**
    - Add sticky notes to document:
      - setup steps
      - onboarding branch
      - daily reminder branch
      - weekly AI digest branch

28. **Test the onboarding path**
    - Submit the form with your own email.
    - Confirm:
      - row added to `Subscribers`
      - welcome email received
      - dates saved in expected format

29. **Test the daily reminder path**
    - Modify one test subscriber’s dates so today matches one of the event days.
    - Run the daily branch manually.
    - Confirm:
      - reminder email sent
      - `last_email_sent` updated
      - row appended to `Send Log`

30. **Test duplicate prevention**
    - Re-run the daily branch without changing dates.
    - Confirm the same reminder is not sent again for that same cycle marker.

31. **Test the weekly digest path**
    - Run the weekly branch manually.
    - Confirm:
      - GPT response parses correctly
      - email formatting is valid
      - log row is appended

32. **Recommended hardening before production**
    - Validate cycle length ranges, for example 20–40 days
    - Escape user-provided HTML values
    - Add an unsubscribe mechanism that really updates `active`
    - Add error handling for invalid AI JSON
    - Add cycle rollover logic so `next_period` and related dates update each cycle

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Weekly Automation: Every woman's cycle is different. This workflow tracks each subscriber's unique cycle, sends perfectly timed reminders before every phase, and delivers personalized weekly wellness tips powered by GPT-4o — all completely automatically. | Overall workflow concept |
| How it works: Subscribers fill in a simple form with their last period date and cycle length. The workflow calculates all key dates — next period, ovulation, fertile window, and PMS start. Every morning it checks all subscribers and sends the right email at the right time. Every Sunday it generates a personalized wellness digest based on each subscriber's current cycle phase. | Overall workflow behavior |
| Setup steps: 1. Create the Google Sheet with 2 sheets — Subscribers and Send Log. 2. Connect Google Sheets, OpenAI, and Gmail credentials. 3. Test by submitting the form with your own email. | Implementation guidance |
| The workflow is currently inactive in the provided export (`"active": false`). | Deployment status |
| The title supplied by the user and the internal workflow name differ. Internal n8n name: `AI-Powered Menstrual Cycle Tracker With Personalized Phase Reminders and Weekly Wellness Coaching`. | Naming context |
| Despite the user title mentioning GPT-4o and Google Sheets, Gmail is the actual email transport used; there is no Outlook integration. | Integration clarification |
| There are multiple entry points: `Cycle Wellness Form`, `Daily 8AM Trigger`, and `Weekly Sunday 9AM Trigger`. | Architecture note |
| No sub-workflow or Execute Workflow node is used anywhere in this workflow. | Architecture note |