Answer new website leads from a webhook with OpenAI, Gmail, Slack and Sheets

https://n8nworkflows.xyz/workflows/answer-new-website-leads-from-a-webhook-with-openai--gmail--slack-and-sheets-20517


# Answer new website leads from a webhook with OpenAI, Gmail, Slack and Sheets

### 1. Workflow Overview

This workflow automates the end-to-end management of inbound website leads. It ingests form submissions via a webhook, uses an AI model to classify and draft personalized communications based on predefined business rules, dispatches immediate responses through email and team messaging, logs the data in a central spreadsheet, and executes a conditional delayed follow-up sequence.

The operational logic is organized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Ingests the raw website form submission and injects global business parameters (services, hours, spreadsheet IDs, and routing channels).
- **1.2 AI Processing & Classification:** Evaluates lead content using an LLM to determine request type, urgency level, and generate structured textual responses.
- **1.3 Immediate Engagement & Logging:** Delivers the automated email response to the customer, notifies the internal team via Slack, and records the interaction in Google Sheets.
- **1.4 Delayed Follow-Up Evaluation:** Pauses execution for two days, retrieves the updated lead state from storage, evaluates whether the lead requires further action, and sends a follow-up email if unresolved.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** This block captures external form payloads via HTTP POST requests and initializes global operational variables required by downstream nodes.
- **Nodes Involved:** `New website enquiry`, `Settings`

- **Node Details:**
  - **`New website enquiry`**
    - **Type & Role:** `n8n-nodes-base.webhook` (Webhook Trigger). Listens for incoming POST requests.
    - **Configuration:** Path set to `new-website-lead`, HTTP method `POST`.
    - **Expressions/Variables:** Outputs standard webhook payload structure (`$json.body.name`, `$json.body.email`, `$json.body.phone`, `$json.body.message`).
    - **Connections:** Input: None (Trigger); Output: `Settings`.
    - **Failure Modes:** Payload format mismatch or missing required body fields can cause evaluation errors in downstream nodes.
  
  - **`Settings`**
    - **Type & Role:** `n8n-nodes-base.set` (Edit Fields). Defines static configuration variables for enterprise context.
    - **Configuration:** Assigns string values for `sheetId`, `businessName`, `services`, `openingHours`, `slackChannel`, and `replyTo`.
    - **Expressions/Variables:** Outputs static property values (e.g., `{{ $json.businessName }}`).
    - **Connections:** Input: `New website enquiry`; Output: `Understand and draft the reply`.
    - **Failure Modes:** Invalid or placeholder IDs (`PASTE_YOUR_GOOGLE_SHEET_ID`) will cause subsequent API calls to fail.

---

#### 2.2 AI Processing & Classification
- **Overview:** Sends the lead context and business parameters to an LLM to categorize the inquiry and generate structured email drafts.
- **Nodes Involved:** `Understand and draft the reply`

- **Node Details:**
  - **`Understand and draft the reply`**
    - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI Advanced AI node). Interfaces with the OpenAI API.
    - **Configuration:** Model set to `gpt-4o-mini`, configured for strict JSON output (`jsonOutput: true`). System prompt injects business parameters (`businessName`, `services`, `openingHours`) and enforces strict constraints (no unverified pricing/promises, word count limits, specific JSON keys: `requestType`, `urgency`, `replySubject`, `replyBody`, `followUpBody`). User prompt injects contact details from the webhook node.
    - **Expressions/Variables:** Uses chained references to access parent scope: `{{ $json.businessName }}`, `{{ $('New website enquiry').item.json.body.name }}`.
    - **Connections:** Input: `Settings`; Output: `Send the reply`.
    - **Failure Modes:** API rate limits, authentication failures, or invalid JSON schema responses from the LLM.

---

#### 2.3 Immediate Engagement & Logging
- **Overview:** Dispatches the drafted email response to the customer, alerts internal staff via team chat, and archives the lead record as "New" in persistent storage.
- **Nodes Involved:** `Send the reply`, `Tell the team in Slack`, `Save the lead`

- **Node Details:**
  - **`Send the reply`**
    - **Type & Role:** `n8n-nodes-base.gmail` (Gmail). Sends an email via Google OAuth2.
    - **Configuration:** Plain text email type (`emailType: text`), appends no attribution, sets custom `replyTo` header.
    - **Expressions/Variables:** `{{ $('New website enquiry').item.json.body.email }}`, `{{ $json.message.content.replyBody }}`, `{{ $json.message.content.replySubject }}`.
    - **Connections:** Input: `Understand and draft the reply`; Output: `Tell the team in Slack`.
    - **Failure Modes:** OAuth2 token expiration, invalid recipient email address, or exceeding Gmail sending quotas.

  - **`Tell the team in Slack`**
    - **Type & Role:** `n8n-nodes-base.slack` (Slack). Posts a notification message to a designated channel.
    - **Configuration:** Posts text messages to a channel resolved dynamically by name.
    - **Expressions/Variables:** `{{ $('Settings').item.json.slackChannel }}`, `{{ $('Understand and draft the reply').item.json.message.content.urgency }}`.
    - **Connections:** Input: `Send the reply`; Output: `Save the lead`.
    - **Failure Modes:** Missing Slack credentials, invalid channel name, or bot lacking permissions to post in the target channel.

  - **`Save the lead`**
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Google Sheets). Appends a new row to a spreadsheet.
    - **Configuration:** Operation set to `append`, mapping columns explicitly (`Date`, `Name`, `Email`, `Phone`, `Message`, `Type`, `Urgency`, `Status`), with `Status` hardcoded to `New`.
    - **Expressions/Variables:** Uses `$now.toISODate()` for dates and evaluates upstream JSON paths.
    - **Connections:** Input: `Tell the team in Slack`; Output: `Wait two days`.
    - **Failure Modes:** Sheet schema mismatch, incorrect Spreadsheet ID, or insufficient Google Drive/Sheets permissions.

---

#### 2.4 Delayed Follow-Up Evaluation
- **Overview:** Pauses execution to allow time for human intervention, re-queries storage after the delay, and conditionally sends a secondary touchpoint if the lead status remains unmodified.
- **Nodes Involved:** `Wait two days`, `Look up the lead again`, `Still marked New?`, `Send one follow up`

- **Node Details:**
  - **`Wait two days`**
    - **Type & Role:** `n8n-nodes-base.wait` (Wait). Pauses workflow execution using resume webhooks or internal scheduling.
    - **Configuration:** Amount: `2`, Unit: `days`.
    - **Expressions/Variables:** None.
    - **Connections:** Input: `Save the lead`; Output: `Look up the lead again`.
    - **Failure Modes:** Workflow inactivation during the waiting period or internal queue storage clearing.

  - **`Look up the lead again`**
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Google Sheets). Performs a lookup query against stored records.
    - **Configuration:** Filters rows where `Email` matches the lead's email address.
    - **Expressions/Variables:** `{{ $('New website enquiry').item.json.body.email }}`.
    - **Connections:** Input: `Wait two days`; Output: `Still marked New?`.
    - **Failure Modes:** Email address not found in sheet due to row deletion or modification.

  - **`Still marked New?`**
    - **Type & Role:** `n8n-nodes-base.if` (If). Evaluates conditional expressions to branch workflow execution.
    - **Configuration:** Checks if the column value equals a specific string.
    - **Expressions/Variables:** Left value: `{{ $json.Status }}`, Operator: `equals`, Right value: `New`.
    - **Connections:** Input: `Look up the lead again`; Output (True): `Send one follow up`.
    - **Failure Modes:** Type mismatch or unexpected schema output from the Google Sheets lookup node.

  - **`Send one follow up`**
    - **Type & Role:** `n8n-nodes-base.gmail` (Gmail). Sends a secondary follow-up email.
    - **Configuration:** Plain text email format, custom subject prefix (`Re: `).
    - **Expressions/Variables:** `{{ $json.Email }}`, `{{ $('Understand and draft the reply').item.json.message.content.followUpBody }}`.
    - **Connections:** Input: `Still marked New? (True)`; Output: None (Terminal node).
    - **Failure Modes:** Authentication expiration or rate-limiting.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | `n8n-nodes-base.stickyNote` | Visual documentation and setup guide. | None | None | ## Answer new website leads in a minute<br><br>### How it works<br>When someone sends your website form, the webhook passes the enquiry to OpenAI. It labels the request type and urgency and drafts a reply using the services and hours in the Settings node. Gmail sends the reply, Slack tells your team, and the lead is saved in Google Sheets. After two days, leads still marked New get one follow up.<br><br>### Setup<br>1. Send your website form to the webhook address in the first node. It expects the fields name, email, phone and message.<br>2. Create a Google Sheet with a tab named Leads and the columns Date, Name, Email, Phone, Message, Type, Urgency, Status.<br>3. Add your OpenAI, Gmail, Slack and Google Sheets credentials to the nodes.<br>4. Fill in the Settings node: sheet ID, business name, services, hours, Slack channel.<br>5. Send a test enquiry, then activate.<br><br>### Customization tips<br>Change the follow up delay in the Wait node or add a text message for urgent requests. |
| **Section: receive** | `n8n-nodes-base.stickyNote` | Visual section boundary marker. | None | None | ## Receive the enquiry<br>Your website form posts here. |
| **Section: understand** | `n8n-nodes-base.stickyNote` | Visual section boundary marker. | None | None | ## Understand and draft the reply<br>Labels the request and writes a personal answer. |
| **Section: reply** | `n8n-nodes-base.stickyNote` | Visual section boundary marker. | None | None | ## Reply and alert the team<br>The customer hears back in a minute and your team sees it in Slack. |
| **Section: follow up** | `n8n-nodes-base.stickyNote` | Visual section boundary marker. | None | None | ## Log and follow up<br>Saves the lead, then nudges once if it is still New after two days. |
| **New website enquiry** | `n8n-nodes-base.webhook` | Ingests incoming form data via HTTP POST. | None | Settings | |
| **Settings** | `n8n-nodes-base.set` | Sets global business parameters and configuration variables. | New website enquiry | Understand and draft the reply | |
| **Understand and draft the reply** | `@n8n/n8n-nodes-langchain.openAi` | Analyzes lead intent, urgency, and drafts email content via LLM. | Settings | Send the reply | |
| **Send the reply** | `n8n-nodes-base.gmail` | Sends the initial AI-generated email reply to the lead. | Understand and draft the reply | Tell the team in Slack | |
| **Tell the team in Slack** | `n8n-nodes-base.slack` | Notifies internal staff channels of the new lead. | Send the reply | Save the lead | |
| **Save the lead** | `n8n-nodes-base.googleSheets` | Appends lead record and initial status to Google Sheets. | Tell the team in Slack | Wait two days | |
| **Wait two days** | `n8n-nodes-base.wait` | Pauses execution for a 48-hour delay window. | Save the lead | Look up the lead again | |
| **Look up the lead again** | `n8n-nodes-base.googleSheets` | Retrieves the current lead record from storage by email filter. | Wait two days | Still marked New? | |
| **Still marked New?** | `n8n-nodes-base.if` | Evaluates if the lead status remains unmodified. | Look up the lead again | Send one follow up | |
| **Send one follow up** | `n8n-nodes-base.gmail` | Sends a secondary follow-up email if the lead status is unresolved. | Still marked New? | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow inside the n8n canvas:

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node. Set the node name to `New website enquiry`.
   - Configure **HTTP Method** to `POST` and set the path to `new-website-lead`.
2. **Create the Configuration Node:**
   - Add a **Set (Edit Fields)** node named `Settings`.
   - Connect `New website enquiry` (Main output) to `Settings` (Main input).
   - Add string assignments:
     - `sheetId`: `PASTE_YOUR_GOOGLE_SHEET_ID`
     - `businessName`: `Your Business Name`
     - `services`: `List your services here, for example: plumbing repairs, water heaters, drain cleaning`
     - `openingHours`: `Monday to Friday, 8 am to 6 pm`
     - `slackChannel`: `new-leads`
     - `replyTo`: `user@example.com`
3. **Add the OpenAI Processing Node:**
   - Add an **OpenAI** (Advanced AI) node named `Understand and draft the reply`.
   - Connect `Settings` output to this node.
   - Configure model selection to `gpt-4o-mini` and enable JSON output mode (`jsonOutput: true`).
   - Populate the system prompt with instructions forcing output into keys: `requestType`, `urgency`, `replySubject`, `replyBody`, and `followUpBody`. Reference business parameters using expressions (e.g., `{{ $json.businessName }}`).
   - Populate the user prompt with webhook data fields: `name`, `email`, `phone`, and `message`.
4. **Add the Gmail Initial Response Node:**
   - Add a **Gmail** node named `Send the reply`.
   - Connect `Understand and draft the reply` to this node.
   - Configure authentication using a Gmail OAuth2 credential.
   - Set **Resource** to `Message`, **Operation** to `Send`. Set **Email Type** to `text`.
   - Map parameters:
     - **Send To**: `{{ $('New website enquiry').item.json.body.email }}`
     - **Subject**: `={{ $json.message.content.replySubject }}`
     - **Message**: `={{ $json.message.content.replyBody }}`
     - **Options > Reply To**: `{{ $('Settings').item.json.replyTo }}`
5. **Add the Slack Notification Node:**
   - Add a **Slack** node named `Tell the team in Slack`.
   - Connect `Send the reply` to this node.
   - Configure a Slack API credential.
   - Set resource to `Message` and operation to `Send`.
   - Configure channel selection by name using the expression: `{{ $('Settings').item.json.slackChannel }}`.
   - Set the message text body to include lead details, detected urgency, and request type.
6. **Add the Google Sheets Logging Node:**
   - Add a **Google Sheets** node named `Save the lead`.
   - Connect `Tell the team in Slack` to this node.
   - Configure Google Sheets OAuth2 credentials.
   - Set operation to `Append`, target document ID to `{{ $('Settings').item.json.sheetId }}` and sheet name to `Leads`.
   - Map columns explicitly:
     - `Date`: `={{ $now.toISODate() }}`
     - `Name`: `={{ $('New website enquiry').item.json.body.name }}`
     - `Email`: `={{ $('New website enquiry').item.json.body.email }}`
     - `Phone`: `={{ $('New website enquiry').item.json.body.phone }}`
     - `Message`: `={{ $('New website enquiry').item.json.body.message }}`
     - `Type`: `={{ $('Understand and draft the reply').item.json.message.content.requestType }}`
     - `Urgency`: `={{ $('Understand and draft the reply').item.json.message.content.urgency }}`
     - `Status`: `New`
7. **Configure the Wait Delay Node:**
   - Add a **Wait** node named `Wait two days`.
   - Connect `Save the lead` to this node.
   - Set **Unit** to `days` and **Amount** to `2`.
8. **Configure the Lead Re-Lookup Node:**
   - Add a **Google Sheets** node named `Look up the lead again`.
   - Connect `Wait two days` to this node.
   - Set operation to `Get Many` or configure **Filters UI**:
     - Lookup Column: `Email`
     - Lookup Value: `{{ $('New website enquiry').item.json.body.email }}`
     - Document ID: `={{ $('Settings').item.json.sheetId }}`
     - Sheet Name: `Leads`
9. **Add the Conditional Branch Node:**
   - Add an **If** node named `Still marked New?`.
   - Connect `Look up the lead again` to this node.
   - Configure a rule where Left Value equals `{{ $json.Status }}` and Right Value equals `New`.
10. **Add the Terminal Follow-Up Email Node:**
    - Add a **Gmail** node named `Send one follow up`.
    - Connect the `True` output of `Still marked New?` to this node.
    - Map parameters:
      - **Send To**: `{{ $json.Email }}`
      - **Subject**: `=Re: {{ $('Understand and draft the reply').item.json.message.content.replySubject }}`
      - **Message**: `={{ $('Understand and draft the reply').item.json.message.content.followUpBody }}`
      - **Options > Reply To**: `={{ $('Settings').item.json.replyTo }}`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure the target Google Sheet contains an exact sheet tab named `Leads` matching the required column headers (`Date`, `Name`, `Email`, `Phone`, `Message`, `Type`, `Urgency`, `Status`) prior to execution. | Sheet Schema Configuration |
| Verify that OAuth2 tokens for Gmail, Slack, and Google Sheets possess proper scopes and remain active to prevent authorization failures during automated delayed steps. | Authentication Management |