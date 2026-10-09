Respond to real estate leads with Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/respond-to-real-estate-leads-with-gmail-and-google-sheets-20566


# Respond to real estate leads with Gmail and Google Sheets

### 1. Workflow Overview

This workflow automates the real estate lead qualification, storage, and response process. Designed for 24/7 responsiveness, it captures inbound enquiries via an HTTP webhook, sanitizes and scores them on a 0–100 scale, logs them in a Google Sheets database (with duplicate prevention using a stable ID), sends a personalized reply to the prospective buyer, seller, or renter, and alerts agents immediately if the lead is designated as "hot."

The workflow logic is grouped into three main functional blocks:
- **1.1 Receive & Qualify:** Ingestion of raw payload, configuration loading, deep normalization, validation, and intent-based lead scoring.
- **1.2 Save to CRM Sheet:** Validation routing, rejection handling for invalid payloads, and upserting lead data into Google Sheets.
- **1.3 Reply & Alert:** Conditional verification of email presence, automated dispatch of tailored client responses, conditional internal agent alerts for high-value prospects, and closing HTTP response handling.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Receive & Qualify

##### Overview
This block acts as the entry point for raw lead data, loads central environment parameters, and executes a comprehensive JavaScript processing function to normalize arbitrary field names, extract buyer/seller metrics, calculate lead temperature, and validate contact details.

##### Nodes Involved
- `Receive lead (webhook)`
- `Config`
- `Normalize and score lead`

##### Node Details

###### `Receive lead (webhook)`
- **Type and Technical Role:** `n8n-nodes-base.webhook` (Trigger node). Listens for incoming HTTP POST requests containing lead submissions.
- **Configuration Choices:** Configured to listen on path `real-estate-lead` with HTTP method `POST`. Response mode is set to handle responses via downstream response nodes.
- **Key Expressions or Variables:** Webhook ID: `aef03947-5b46-55cf-b89e-19b69bc8e179`.
- **Input and Output Connections:** Input: External HTTP POST request; Output: Connected to `Config`.
- **Version-Specific Requirements:** Type version 2.1.
- **Edge Cases or Potential Failure Types:** Payload parsing errors if malformed JSON is posted; unhandled HTTP methods (GET instead of POST) will return 404/405 status codes.

###### `Config`
- **Type and Technical Role:** `n8n-nodes-base.set` (Data transformation/storage node). Defines global variables utilized across downstream nodes and code blocks.
- **Configuration Choices:** Sets static parameters including `googleSheetId`, `agencyName`, `agentName`, `agentEmail`, `agentPhone`, `bookingLink`, `currencySymbol`, `hotScoreThreshold` (70), and `warmScoreThreshold` (40).
- **Key Expressions or Variables:** Static string and number assignments.
- **Input and Output Connections:** Input: `Receive lead (webhook)`; Output: `Normalize and score lead`.
- **Version-Specific Requirements:** Type version 3.5.
- **Edge Cases or Potential Failure Types:** Placeholder values (e.g., `PASTE_YOUR_GOOGLE_SHEET_ID`, `user@example.com`) must be replaced, or downstream API integrations will fail.

###### `Normalize and score lead`
- **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript execution node). Normalizes inconsistent field names (English and French aliases), computes lead intention (buy, sell, rent, other), parses budgets and timelines, generates a stable cryptographic Lead ID (FNV-1a hash), calculates a 0–100 score, and builds template objects for emails and web alerts.
- **Configuration Choices:** Executes custom ES6 JavaScript logic utilizing internal string-normalization and heuristic parsing functions.
- **Key Expressions or Variables:** Reads configuration variables via `$(T.nodes.config).first().json` and inbound payload properties via `$(T.nodes.leadWebhook).first().json.body`.
- **Input and Output Connections:** Input: `Config`; Output: `Is the lead valid?`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases or Potential Failure Types:** JavaScript runtime errors if incoming payloads deviate significantly from expected structures, or if the preceding nodes do not supply valid items.

---

#### Block 1.2: Save to CRM Sheet

##### Overview
This block checks whether the normalized lead contains valid contact details. If invalid, it short-circuits the flow and returns an HTTP 422 error response. If valid, it saves or updates the contact record in a Google Sheets database using a stable ID key.

##### Nodes Involved
- `Is the lead valid?`
- `Reject invalid lead`
- `Save lead to Google Sheets`

##### Node Details

###### `Is the lead valid?`
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow control node). Evaluates the boolean validity flag produced by the normalization script.
- **Configuration Choices:** Checks whether `{{ $json.isValid }}` evaluates to `true`.
- **Key Expressions or Variables:** `={{ $json.isValid }}`
- **Input and Output Connections:** Input: `Normalize and score lead`; Outputs: True branch connects to `Save lead to Google Sheets`, False branch connects to `Reject invalid lead`.
- **Version-Specific Requirements:** Type version 2.3.
- **Edge Cases or Potential Failure Types:** Loose type validation enabled; unexpected data types could route incorrectly if schema definitions change.

###### `Reject invalid lead`
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook response node). Returns an immediate HTTP error response when lead data fails validation.
- **Configuration Choices:** Response code set to `422 Unprocessable Entity`; responds with JSON containing error details.
- **Key Expressions or Variables:** `={{ JSON.stringify({ ok: false, errors: $json.errors }) }}`
- **Input and Output Connections:** Input: False branch of `Is the lead valid?`; Output: Terminal node.
- **Version-Specific Requirements:** Type version 1.5.
- **Edge Cases or Potential Failure Types:** Attempting to respond to a webhook connection that has already closed or timed out.

###### `Save lead to Google Sheets`
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (Integration node). Appends a new row or updates an existing record in a specified Google Sheet.
- **Configuration Choices:** Resource: `sheet`, Operation: `appendOrUpdate`, Mapping mode: `defineBelow`, Matching columns: `Lead ID`. Sheet name set to `Leads`.
- **Key Expressions or Variables:** Document ID: `={{ $('Config').first().json.googleSheetId }}`. Column mappings map directly to properties within `{{ $json.lead[...] }}`.
- **Input and Output Connections:** Input: True branch of `Is the lead valid?`; Output: `Lead has an email?`.
- **Version-Specific Requirements:** Type version 4.7.
- **Edge Cases or Potential Failure Types:** Google OAuth2 authentication expiration; missing or renamed columns in the target Google Sheet tab; quota limits exceeded on Google Sheets API.

---

#### Block 1.3: Reply & Alert

##### Overview
This block manages outbound communications. It verifies if an email address is present to dispatch an instant personalized acknowledgment to the lead via Gmail, checks if the lead's score meets the hot threshold to notify the agent via email, and finally returns a successful JSON confirmation payload to the calling client.

##### Nodes Involved
- `Lead has an email?`
- `Send instant reply to lead`
- `Is it a hot lead?`
- `Alert agent about hot lead`
- `Confirm lead received`

##### Node Details

###### `Lead has an email?`
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow control node). Determines whether the lead profile includes a usable email address before attempting email dispatch.
- **Configuration Choices:** Evaluates the `hasEmail` flag generated during normalization.
- **Key Expressions or Variables:** `={{ $("Normalize and score lead").first().json.hasEmail }}`
- **Input and Output Connections:** Input: `Save lead to Google Sheets`; Outputs: True branch connects to `Send instant reply to lead`, False branch connects directly to `Is it a hot lead?`.
- **Version-Specific Requirements:** Type version 2.3.
- **Edge Cases or Potential Failure Types:** Malformed or missing boolean properties will evaluate using loose type conditions.

###### `Send instant reply to lead`
- **Type and Technical Role:** `n8n-nodes-base.gmail` (Integration node). Sends an HTML-formatted confirmation email directly to the prospective client.
- **Configuration Choices:** Resource: `message`, Operation: `send`, Email type: `html`. Options configure the custom sender name and suppress automated n8n attributions.
- **Key Expressions or Variables:** 
  - Send To: `={{ $("Normalize and score lead").first().json.reply.to }}`
  - Subject: `={{ $("Normalize and score lead").first().json.reply.subject }}`
  - Message: `={{ $("Normalize and score lead").first().json.reply.html }}`
  - Sender Name: `={{ $('Config').first().json.agencyName }}`
- **Input and Output Connections:** Input: True branch of `Lead has an email?`; Output: `Is it a hot lead?`.
- **Version-Specific Requirements:** Type version 2.2.
- **Edge Cases or Potential Failure Types:** Gmail API rate limiting; invalid recipient email format; OAuth token revocation.

###### `Is it a hot lead?`
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow control node). Evaluates whether the calculated lead score crosses the configured hot threshold.
- **Configuration Choices:** Checks whether the lead is flagged as hot.
- **Key Expressions or Variables:** `={{ $("Normalize and score lead").first().json.isHot }}`
- **Input and Output Connections:** Inputs: False branch of `Lead has an email?` or output of `Send instant reply to lead`; Outputs: True branch connects to `Alert agent about hot lead`, False branch connects to `Confirm lead received`.
- **Version-Specific Requirements:** Type version 2.3.
- **Edge Cases or Potential Failure Types:** Threshold mismatches if configuration values are altered without updating scoring logic boundaries.

###### `Alert agent about hot lead`
- **Type and Technical Role:** `n8n-nodes-base.gmail` (Integration node). Dispatches an immediate high-priority alert email to the real estate agent for high-scoring prospects.
- **Configuration Choices:** Resource: `message`, Operation: `send`, Email type: `html`.
- **Key Expressions or Variables:**
  - Send To: `={{ $('Config').first().json.agentEmail }}`
  - Subject: `={{ $("Normalize and score lead").first().json.alert.subject }}`
  - Message: `={{ $("Normalize and score lead").first().json.alert.html }}`
  - Sender Name: `={{ $('Config').first().json.agencyName }}`
- **Input and Output Connections:** Input: True branch of `Is it a hot lead?`; Output: `Confirm lead received`.
- **Version-Specific Requirements:** Type version 2.2.
- **Edge Cases or Potential Failure Types:** Agent email misconfiguration; delivery failures due to spam filtering on internal mail servers.

###### `Confirm lead received`
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook response node). Returns a successful HTTP 200 response payload back to the webhook caller, containing lead summary data.
- **Configuration Choices:** Response code set to `200 OK`, responding with formatted JSON.
- **Key Expressions or Variables:** `={{ JSON.stringify({ ok: true, leadId: $("Normalize and score lead").first().json.leadId, temperature: $("Normalize and score lead").first().json.temperature, score: $("Normalize and score lead").first().json.score }) }}`
- **Input and Output Connections:** Inputs: False branch of `Is it a hot lead?` or output of `Alert agent about hot lead`; Output: Terminal node.
- **Version-Specific Requirements:** Type version 1.5.
- **Edge Cases or Potential Failure Types:** Webhook connection timeout if prior email nodes experience high latency.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Receive lead (webhook)` | `n8n-nodes-base.webhook` | Ingests inbound HTTP POST lead data | None | `Config` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 1. Receive & qualify<br>Normalizes any form payload, validates it and scores the lead. |
| `Config` | `n8n-nodes-base.set` | Sets global configuration variables | `Receive lead (webhook)` | `Normalize and score lead` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 1. Receive & qualify<br>Normalizes any form payload, validates it and scores the lead. |
| `Normalize and score lead` | `n8n-nodes-base.code` | Normalizes, validates, scores lead data | `Config` | `Is the lead valid?` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 1. Receive & qualify<br>Normalizes any form payload, validates it and scores the lead. |
| `Is the lead valid?` | `n8n-nodes-base.if` | Evaluates lead validity flag | `Normalize and score lead` | `Save lead to Google Sheets`, `Reject invalid lead` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 1. Receive & qualify<br>Normalizes any form payload, validates it and scores the lead. |
| `Reject invalid lead` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 422 for invalid payloads | `Is the lead valid?` (False) | None | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 1. Receive & qualify<br>Normalizes any form payload, validates it and scores the lead. |
| `Save lead to Google Sheets` | `n8n-nodes-base.googleSheets` | Appends or updates lead data in Google Sheets | `Is the lead valid?` (True) | `Lead has an email?` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 2. Save to your CRM sheet<br>New leads are added, repeat enquiries update the same row. |
| `Lead has an email?` | `n8n-nodes-base.if` | Verifies presence of a valid email address | `Save lead to Google Sheets` | `Send instant reply to lead`, `Is it a hot lead?` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 3. Reply & alert<br>Instant personalized email, hot-lead alert, then the form gets its answer. |
| `Send instant reply to lead` | `n8n-nodes-base.gmail` | Sends personalized reply to lead | `Lead has an email?` (True) | `Is it a hot lead?` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 3. Reply & alert<br>Instant personalized email, hot-lead alert, then the form gets its answer. |
| `Is it a hot lead?` | `n8n-nodes-base.if` | Evaluates if score meets hot threshold | `Lead has an email?` (False), `Send instant reply to lead` | `Alert agent about hot lead`, `Confirm lead received` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config** node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 3. Reply & alert<br>Instant personalized email, hot-lead alert, then the form gets its answer. |
| `Alert agent about hot lead` | `n8n-nodes-base.gmail` | Sends hot-lead notification to agent | `Is it a hot lead?` (True) | `Confirm lead received` | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config` node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 3. Reply & alert<br>Instant personalized email, hot-lead alert, then the form gets its answer. |
| `Confirm lead received` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 200 JSON success response | `Is it a hot lead?` (False), `Alert agent about hot lead` | None | ## Respond instantly to real estate leads<br><br>Speed to lead wins listings and sales: this workflow answers every enquiry within seconds, 24/7, and tells you which ones to call first.<br><br>### How it works<br>1. Your website form, portal parser or Zapier-style hook posts the enquiry to the **webhook**.<br>2. The lead is normalized (any field names), validated and **scored 0-100** — phone, timeline, budget, property reference and financing keywords raise the score.<br>3. It is saved in Google Sheets; repeat enquiries update the same row (stable Lead ID).<br>4. The lead gets a personalized email for buying, selling or renting, with your booking link.<br>5. **Hot** leads trigger an instant alert to you. The form receives a JSON answer.<br><br>### Setup<br>- Connect **Gmail** and **Google Sheets** credentials.<br>- Fill the **Config` node: Sheet ID, agency, name, email, phone, booking link, score thresholds.<br>- Uses one Google Sheet with three tabs — **Leads**, **Listings**, **Viewings** — shared by the whole Real Estate Automation Kit (column list in the README).<br>- Point your form to the production webhook URL.<br><br>### Customization tips<br>- Edit the email texts and scoring weights in **Normalize and score lead**.<br>- Swap the agent alert for Slack, Telegram or Twilio SMS.<br><br>## 3. Reply & alert<br>Instant personalized email, hot-lead alert, then the form gets its answer. |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the workflow manually in n8n:

1. **Create the Webhook Trigger Node**
   - Add a **Webhook** node named `Receive lead (webhook)`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `real-estate-lead`.
   - Set **Response Mode** to `Response Node`.

2. **Create the Config Set Node**
   - Add a **Set** node named `Config`.
   - Connect `Receive lead (webhook)` output to `Config`.
   - Add the following string and number assignments:
     - `googleSheetId`: `PAStransE_YOUR_GOOGLE_SHEET_ID` (string)
     - `agencyName`: `Your Agency` (string)
     - `agentName`: `Your Name` (string)
     - `agentEmail`: `user@example.com` (string)
     - `agentPhone`: `+1234567890` (string)
     - `bookingLink`: `https://calendly.com/your-link` (string)
     - `currencySymbol`: `$` (string)
     - `hotScoreThreshold`: `70` (number)
     - `warmScoreThreshold`: `40` (number)

3. **Create the Code Node for Normalization and Scoring**
   - Add a **Code** node named `Normalize and score lead`.
   - Connect `Config` output to this node.
   - Insert the JavaScript ES6 processing code provided in the source workflow payload, ensuring helper functions (`esc`, `fill`, `greet`, `keyOf`, `money`, `isEmail`, `parseMoney`, `classify`) and configuration references remain intact.

4. **Create the Validity Evaluation IF Node**
   - Add an **If** node named `Is the lead valid?`.
   - Connect `Normalize and score lead` output to this node.
   - Configure condition: Left Value `={{ $json.isValid }}`, Operator `true`.

5. **Create the Webhook Rejection Node**
   - Add a **Respond to Webhook** node named `Reject invalid lead`.
   - Connect the **false** output branch of `Is the lead valid?` to this node.
   - Set **Response Code** to `422`.
   - Set **Response Body** expression to `={{ JSON.stringify({ ok: false, errors: $json.errors }) }}`.

6. **Create the Google Sheets Integration Node**
   - Add a **Google Sheets** node named `Save lead to Google Sheets`.
   - Connect the **true** output branch of `Is the lead valid?` to this node.
   - Configure **Resource**: `Sheet`, **Operation**: `Append or Update`.
   - Set **Document ID** to `={{ $('Config').first().json.googleSheetId }}` and **Sheet Name** to `Leads`.
   - Set **Mapping Mode** to `Define Below` and **Matching Columns** to `Lead ID`. Map each sheet column (`Name`, `Email`, `Phone`, `Score`, `Budget`, `Intent`, `Source`, `Lead ID`, `Message`, `Timeline`, `Temperature`, `Min Bedrooms`, `Property Ref`, `Property Type`, `Follow-up Step`, `Last Contact At`, `Last Enquiry At`, `Preferred Areas`) to the corresponding property in `{{ $json.lead[...] }}`.
   - Configure Google Sheets OAuth2 credentials.

7. **Create the Email Presence Check IF Node**
   - Add an **If** node named `Lead has an email?`.
   - Connect `Save lead to Google Sheets` output to this node.
   - Configure condition: Left Value `={{ $("Normalize and score lead").first().json.hasEmail }}`, Operator `true`.

8. **Create the Instant Lead Reply Gmail Node**
   - Add a **Gmail** node named `Send instant reply to lead`.
   - Connect the **true** output branch of `Lead has an email?` to this node.
   - Configure **Resource**: `Message`, **Operation**: `Send`, **Email Type**: `HTML`.
   - Set **Send To** to `={{ $("Normalize and score lead").first().json.reply.to }}`.
   - Set **Subject** to `={{ $("Normalize and score lead").first().json.reply.subject }}`.
   - Set **Message** to `={{ $("Normalize and score lead").first().json.reply.html }}`.
   - In **Options**, set **Sender Name** to `={{ $('Config').first().json.agencyName }}` and **Append Attribution** to `false`.
   - Configure Gmail OAuth2 credentials.

9. **Create the Hot Lead Evaluation IF Node**
   - Add an **If** node named `Is it a hot lead?`.
   - Connect the **false** output branch of `Lead has an email?` and the output of `Send instant reply to lead` to this node.
   - Configure condition: Left Value `={{ $("Normalize and score lead").first().json.isHot }}`, Operator `true`.

10. **Create the Agent Alert Gmail Node**
    - Add a **Gmail** node named `Alert agent about hot lead`.
    - Connect the **true** output branch of `Is it a hot lead?` to this node.
    - Configure **Resource**: `Message`, **Operation**: `Send`, **Email Type**: `HTML`.
    - Set **Send To** to `={{ $('Config').first().json.agentEmail }}`.
    - Set **Subject** to `={{ $("Normalize and score lead").first().json.alert.subject }}`.
    - Set **Message** to `={{ $("Normalize and score lead").first().json.alert.html }}`.
    - In **Options**, set **Sender Name** to `={{ $('Config').first().json.agencyName }}` and **Append Attribution** to `false`.
    - Configure Gmail OAuth2 credentials.

11. **Create the Final Success Webhook Response Node**
    - Add a **Respond to Webhook** node named `Confirm lead received`.
    - Connect the **false** output branch of `Is it a hot lead?` and the output of `Alert agent about hot lead` to this node.
    - Set **Response Code** to `200`.
    - Set **Response Body** expression to `={{ JSON.stringify({ ok: true, leadId: $("Normalize and score lead").first().json.leadId, temperature: $("Normalize and score lead").first().json.temperature, score: $("Normalize and score lead").first().json.score }) }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Real estate lead response automation kit description | Part of an automation framework designed to capture enquiries, update CRM records, and improve conversion speed via instant lead engagement. |
| Google Sheets database structure requirement | Requires a target spreadsheet containing three standard tabs: `Leads`, `Listings`, and `Viewings`. |