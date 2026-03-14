Triage and schedule healthcare appointments with Azure OpenAI, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/triage-and-schedule-healthcare-appointments-with-azure-openai--google-sheets-and-gmail-13960


# Triage and schedule healthcare appointments with Azure OpenAI, Google Sheets and Gmail

# 1. Workflow Overview

This workflow implements a healthcare appointment booking and follow-up system in n8n. It is organized into three parallel automation pipelines:

1. **Patient Intake & Triage**  
   Receives patient lead data through a webhook, formats it, sends it to an Azure OpenAI-powered triage agent, normalizes the AI output, and stores the result in Google Sheets.

2. **Doctor Assignment & Notification**  
   Watches the patient intake sheet for new entries, uses an AI agent plus a Google Sheets tool to select the best doctor, saves the assignment to a scheduling sheet, then prepares and sends an appointment summary email to a doctor.

3. **Post-Visit Feedback**  
   Runs on a schedule, reads completed appointments from Google Sheets, filters eligible rows, and emails patients a feedback request.

## 1.1 Input Reception and Triage
This block starts with an HTTP webhook receiving patient data. The workflow standardizes field names, invokes an AI triage model, and converts the response into a predictable structure for sheet storage.

## 1.2 Persistence of Triage Results
The normalized triage output is written into the **Patient forms** tab in Google Sheets. This sheet acts as the handoff point to the doctor assignment pipeline.

## 1.3 Doctor Assignment from New Sheet Rows
A Google Sheets trigger detects new patient rows. An AI doctor-assignment agent reads the incoming row, consults the **Doctor details** tab as a tool, and returns a structured assignment.

## 1.4 Persistence of Assignment and Doctor Notification
The workflow normalizes the assignment result, writes it into the **Scheduled appointments** tab, reloads appointment data, builds an HTML email summary, and sends it via Gmail.

## 1.5 Scheduled Feedback Outreach
An hourly scheduler reads the appointments sheet, filters rows marked as visited/completed, and sends a text email containing a feedback link.

---

# 2. Block-by-Block Analysis

## Block 1 — Patient Intake & Triage

### Overview
This block receives patient lead data from an external system, reformats the payload into workflow-friendly keys, and sends it to an AI triage agent. The triage agent classifies urgency, suggests a department, and returns JSON that is normalized before storage.

### Nodes Involved
- 🔔 Patient Lead Webhook
- 🗂️ Format Patient Data
- 🤖 AI Triage Agent
- 🧠 Azure OpenAI – Triage Model
- ⚙️ Normalize Triage Output

---

### Node: 🔔 Patient Lead Webhook

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that accepts incoming HTTP POST requests.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `patient-lead`
  - No extra options configured
- **Key expressions or variables used:**  
  None in node config; it passes raw request JSON downstream.
- **Input and output connections:**  
  - Input: none
  - Output: 🗂️ Format Patient Data
- **Version-specific requirements:**  
  Uses Webhook node typeVersion 1.
- **Edge cases or potential failure types:**  
  - Workflow must be active for production webhook URL to work
  - Payload shape must match downstream expressions
  - Missing fields may not break the webhook itself, but will produce blank values later
- **Sub-workflow reference:**  
  None

---

### Node: 🗂️ Format Patient Data

- **Type and technical role:** `n8n-nodes-base.set`  
  Maps incoming webhook fields into standardized keys.
- **Configuration choices:**  
  Creates string fields:
  - `patient_name = {{$json["name"]}}`
  - `phone = {{$json["phone"]}}`
  - `symptoms = {{$json["symptoms"]}}`
  - `age = {{$json["age"]}}`
  - `gender = {{$json["gender"]}}`
  - `additonal notes = {{$json["additonal notes"]}}`
- **Key expressions or variables used:**  
  References raw webhook payload properties.
- **Input and output connections:**  
  - Input: 🔔 Patient Lead Webhook
  - Output: 🤖 AI Triage Agent
- **Version-specific requirements:**  
  Uses Set node typeVersion 2.
- **Edge cases or potential failure types:**  
  - There is a spelling inconsistency: `additonal notes` is misspelled
  - The next AI node references `{{$json.notes}}`, not `{{$json["additonal notes"]}}`; this means additional notes will usually be blank
  - If the webhook sends nested JSON instead of flat keys, expressions will fail silently to empty values
- **Sub-workflow reference:**  
  None

---

### Node: 🤖 AI Triage Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent that performs safe triage and returns structured JSON.
- **Configuration choices:**  
  - Prompt type: defined manually
  - User prompt includes:
    - patient name
    - age
    - gender
    - phone
    - symptoms
    - additional notes
  - System message enforces:
    - non-diagnostic role
    - urgency categories LOW / MEDIUM / HIGH / CRITICAL
    - approved department list
    - JSON-only output format
    - fallback to MEDIUM if unsure
- **Key expressions or variables used:**  
  - `{{$json.patient_name}}`
  - `{{$json.age}}`
  - `{{$json.gender}}`
  - `{{$json.phone}}`
  - `{{$json.symptoms}}`
  - `{{$json.notes}}` ← likely wrong key
- **Input and output connections:**  
  - Main input: 🗂️ Format Patient Data
  - AI language model input: 🧠 Azure OpenAI – Triage Model
  - Main output: ⚙️ Normalize Triage Output
- **Version-specific requirements:**  
  Uses LangChain Agent typeVersion 3. Requires compatible n8n version with AI nodes enabled.
- **Edge cases or potential failure types:**  
  - Azure credential or deployment misconfiguration
  - Non-JSON AI output despite prompt instructions
  - Missing `notes` field because upstream created `additonal notes`
  - Token or rate-limit issues from Azure OpenAI
- **Sub-workflow reference:**  
  None

---

### Node: 🧠 Azure OpenAI – Triage Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`  
  Azure-hosted chat model backing the triage agent.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
  - No advanced options set
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output (AI language model): 🤖 AI Triage Agent
- **Version-specific requirements:**  
  Uses typeVersion 1. Requires valid Azure OpenAI credentials with endpoint, API key, and deployment support.
- **Edge cases or potential failure types:**  
  - Invalid deployment name
  - Unsupported model in Azure subscription
  - Regional endpoint mismatch
  - Authentication failure
- **Sub-workflow reference:**  
  None

---

### Node: ⚙️ Normalize Triage Output

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses and standardizes the AI response into a clean object and a sheet-ready row.
- **Configuration choices:**  
  The JavaScript:
  - Tries to parse AI output if returned as a string
  - Falls back to `JSON.parse(raw.output || "{}")`
  - Normalizes urgency into one of:
    - `CRITICAL`
    - `HIGH`
    - `MEDIUM`
    - `LOW`
  - Maps urgency to appointment priority:
    - CRITICAL → `Immediate`
    - HIGH → `Same Day`
    - MEDIUM → `24-48 Hours`
    - LOW → `3-5 Days`
  - Defaults department to `General Physician`
  - Builds `sheet_row` object with columns:
    - Name
    - Age
    - Gender
    - Phone
    - Symptoms
    - Urgency
    - Department
    - Priority
    - Recommended Action
- **Key expressions or variables used:**  
  Operates on incoming `$json`
- **Input and output connections:**  
  - Input: 🤖 AI Triage Agent
  - Output: 📝 Save Triage to Patient Forms Sheet
- **Version-specific requirements:**  
  Code node typeVersion 2.
- **Edge cases or potential failure types:**  
  - If the AI returns malformed JSON not recoverable by the parsing logic, the node can throw
  - If the agent wraps JSON differently than expected, normalization may produce partial blanks
  - No validation that required sheet fields are present
- **Sub-workflow reference:**  
  None

---

## Block 2 — Save Triage to Google Sheets

### Overview
This block persists the normalized triage result into the intake sheet. It acts as the bridge between patient submission and doctor assignment.

### Nodes Involved
- 📝 Save Triage to Patient Forms Sheet

---

### Node: 📝 Save Triage to Patient Forms Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Writes triage data to Google Sheets.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Document: spreadsheet named **Patient & Doctors details**
  - Sheet: **Patient forms**
  - Mapping mode: auto-map input data
  - No explicit matching columns configured
- **Key expressions or variables used:**  
  None directly; uses incoming normalized JSON.
- **Input and output connections:**  
  - Input: ⚙️ Normalize Triage Output
  - Output: none
- **Version-specific requirements:**  
  Google Sheets node typeVersion 4.7 with OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - `appendOrUpdate` without matching columns can behave unexpectedly depending on sheet structure
  - Sheet headers must match incoming keys or nested object handling may not align as expected
  - Google API quota/auth issues
  - The nested `sheet_row` object may not be written as intended unless headers and mapping are compatible
- **Sub-workflow reference:**  
  None

---

## Block 3 — Doctor Assignment

### Overview
This block is triggered when a new intake row appears in Google Sheets. It passes patient data to an AI assignment agent, which consults the doctor list via a Google Sheets tool and chooses the most suitable doctor.

### Nodes Involved
- 📊 New Patient Row Trigger
- 🤖 AI Doctor Assignment Agent
- 🧠 Azure OpenAI – Assignment Model
- 📋 Fetch Doctor List from Sheet
- ⚙️ Normalize Assignment Output

---

### Node: 📊 New Patient Row Trigger

- **Type and technical role:** `n8n-nodes-base.googleSheetsTrigger`  
  Poll-based trigger that detects new rows in the intake sheet.
- **Configuration choices:**  
  - Polling interval: every minute
  - Document: **Patient & Doctors details**
  - Sheet: **Patient forms**
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: none
  - Output: 🤖 AI Doctor Assignment Agent
- **Version-specific requirements:**  
  Trigger node typeVersion 1 with Google Sheets Trigger OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Polling delays up to a minute
  - Duplicate processing if trigger state resets
  - Header changes may affect emitted field names and downstream expressions
- **Sub-workflow reference:**  
  None

---

### Node: 🤖 AI Doctor Assignment Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI routing agent that assigns a doctor based on symptoms and doctor list data.
- **Configuration choices:**  
  - User prompt includes:
    - `{{$json.name}}`
    - `{{$json.age}}`
    - `{{$json.gender}}`
    - `{{$json.symptoms}}`
  - System message enforces:
    - mandatory use of doctor sheet tool
    - specialty mapping rules
    - assignment only to doctors present in sheet
    - JSON-only output
- **Key expressions or variables used:**  
  - `{{$json.name}}`
  - `{{$json.age}}`
  - `{{$json.gender}}`
  - `{{$json.symptoms}}`
- **Input and output connections:**  
  - Main input: 📊 New Patient Row Trigger
  - AI language model input: 🧠 Azure OpenAI – Assignment Model
  - AI tool input: 📋 Fetch Doctor List from Sheet
  - Main output: ⚙️ Normalize Assignment Output
- **Version-specific requirements:**  
  LangChain Agent typeVersion 3 with AI tool support.
- **Edge cases or potential failure types:**  
  - Field-name mismatch risk: the patient intake sheet appears to use columns like `Name`, `Age`, `Gender`, `Symptoms`, while this prompt expects lowercase `name`, `age`, `gender`, `symptoms`
  - If trigger output preserves sheet header casing, the agent may receive blanks
  - Tool call failure if Sheets credentials are invalid
  - Non-JSON response from model
- **Sub-workflow reference:**  
  None

---

### Node: 🧠 Azure OpenAI – Assignment Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`  
  Chat model used by the doctor assignment agent.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output (AI language model): 🤖 AI Doctor Assignment Agent
- **Version-specific requirements:**  
  TypeVersion 1 with Azure OpenAI credentials.
- **Edge cases or potential failure types:**  
  Same Azure-related risks as the triage model node.
- **Sub-workflow reference:**  
  None

---

### Node: 📋 Fetch Doctor List from Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheetsTool`  
  AI tool node that exposes doctor data from Google Sheets to the assignment agent.
- **Configuration choices:**  
  - Document: **Patient & Doctors details**
  - Sheet: **Doctor details**
  - No additional filters/options
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output (AI tool): 🤖 AI Doctor Assignment Agent
- **Version-specific requirements:**  
  Google Sheets Tool typeVersion 4.7 and AI-tool-capable n8n version.
- **Edge cases or potential failure types:**  
  - Missing or inconsistent doctor specialty data
  - Empty doctor sheet
  - Tool may return too much data if the sheet grows large
- **Sub-workflow reference:**  
  None

---

### Node: ⚙️ Normalize Assignment Output

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses AI assignment output, standardizes values, and prepares the final row for the appointments sheet.
- **Configuration choices:**  
  The JavaScript:
  - Parses raw JSON or extracts JSON from text
  - Normalizes gender to `Male`, `Female`, or `Other`
  - Converts age to integer when possible
  - Normalizes specialty into one of:
    - Cardiologist
    - Dermatologist
    - Orthopedic
    - Neurologist
    - ENT Specialist
    - Gastroenterologist
    - Pulmonologist
    - Gynecologist
    - Pediatrician
    - default `General Physician`
  - Builds `sheet_row` with columns:
    - Patient Name
    - Age
    - Gender
    - Symptoms
    - Specialty
    - Doctor Assigned
- **Key expressions or variables used:**  
  Operates on incoming `$json`
- **Input and output connections:**  
  - Input: 🤖 AI Doctor Assignment Agent
  - Output: 📝 Save Assignment to Appointments Sheet
- **Version-specific requirements:**  
  Code node typeVersion 2.
- **Edge cases or potential failure types:**  
  - If patient data is blank due to wrong upstream field names, normalization preserves blanks
  - If AI returns invalid structure, defaults may erase meaningful information
- **Sub-workflow reference:**  
  None

---

## Block 4 — Save Assignment and Email the Doctor

### Overview
This block stores assignment results in the scheduling sheet, reads appointment data, converts it into an HTML digest, and emails it through Gmail. It is intended as the operational notification step after assignment.

### Nodes Involved
- 📝 Save Assignment to Appointments Sheet
- 📋 Fetch Appointments for Doctor Email
- ⚙️ Build Doctor Schedule Email HTML
- 📧 Send Schedule Email to Doctor

---

### Node: 📝 Save Assignment to Appointments Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Writes doctor assignment results into the appointments sheet.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Sheet: **Scheduled appointments**
  - Auto-map input data enabled
- **Key expressions or variables used:**  
  None directly
- **Input and output connections:**  
  - Input: ⚙️ Normalize Assignment Output
  - Output: 📋 Fetch Appointments for Doctor Email
- **Version-specific requirements:**  
  Google Sheets node typeVersion 4.7.
- **Edge cases or potential failure types:**  
  - Header mismatch can create write issues
  - Auto-mapping may not flatten `sheet_row` as expected
  - Risk of overwrite or duplicate behavior due to append/update mode without explicit matching columns
- **Sub-workflow reference:**  
  None

---

### Node: 📋 Fetch Appointments for Doctor Email

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the appointments sheet for inclusion in the doctor email digest.
- **Configuration choices:**  
  - Reads from **Scheduled appointments**
  - No filters are configured
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: 📝 Save Assignment to Appointments Sheet
  - Output: ⚙️ Build Doctor Schedule Email HTML
- **Version-specific requirements:**  
  Google Sheets node typeVersion 4.7.
- **Edge cases or potential failure types:**  
  - Because there is no filter by assigned doctor or date, this node may fetch all rows
  - Email content may include unrelated appointments
  - If sheet column names differ from code expectations, HTML fields become blank
- **Sub-workflow reference:**  
  None

---

### Node: ⚙️ Build Doctor Schedule Email HTML

- **Type and technical role:** `n8n-nodes-base.code`  
  Builds a styled HTML table summarizing appointment rows.
- **Configuration choices:**  
  The JavaScript:
  - Reads all incoming items using `$input.all()`
  - Extracts `doctor_name` from the first row, defaulting to `"Doctor"`
  - Uses current local date
  - Builds table rows from fields:
    - `time`
    - `patient_name`
    - `age`
    - `gender`
    - `symptoms`
  - Returns `{ email_html: html }`
- **Key expressions or variables used:**  
  `$input.all()`
- **Input and output connections:**  
  - Input: 📋 Fetch Appointments for Doctor Email
  - Output: 📧 Send Schedule Email to Doctor
- **Version-specific requirements:**  
  Code node typeVersion 2.
- **Edge cases or potential failure types:**  
  - Source sheet likely uses headers such as `Patient Name` and `Doctor Assigned`, not `patient_name` or `doctor_name`
  - This mismatch can yield generic doctor greeting and empty table cells
  - No escaping of HTML-sensitive patient data
- **Sub-workflow reference:**  
  None

---

### Node: 📧 Send Schedule Email to Doctor

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated HTML appointment schedule.
- **Configuration choices:**  
  - Recipient: `info@example.com`
  - Subject: `Today's Appointment Schedule`
  - Message body: `{{$json.email_html}}`
- **Key expressions or variables used:**  
  `{{$json.email_html}}`
- **Input and output connections:**  
  - Input: ⚙️ Build Doctor Schedule Email HTML
  - Output: none
- **Version-specific requirements:**  
  Gmail node typeVersion 2.2 with Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Recipient is static, not dynamic per assigned doctor
  - If Gmail account lacks permission or OAuth token expires, send fails
  - HTML rendering depends on Gmail node handling of message body; no explicit HTML/email-type option is set here
- **Sub-workflow reference:**  
  None

---

## Block 5 — Post-Visit Feedback

### Overview
This block runs every hour, loads appointment data, filters rows with a visit/completion indicator, and sends feedback emails to patients. It is designed as a basic post-appointment outreach loop.

### Nodes Involved
- ⏰ Hourly Feedback Schedule Trigger
- 📋 Fetch All Appointments for Feedback
- 🔀 Filter Completed Visits
- 📧 Send Feedback Email to Patient

---

### Node: ⏰ Hourly Feedback Schedule Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based workflow entry point.
- **Configuration choices:**  
  - Interval rule: every hour
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: none
  - Output: 📋 Fetch All Appointments for Feedback
- **Version-specific requirements:**  
  Schedule Trigger typeVersion 1.3.
- **Edge cases or potential failure types:**  
  - Workflow must be active
  - Timezone behavior depends on instance settings
- **Sub-workflow reference:**  
  None

---

### Node: 📋 Fetch All Appointments for Feedback

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads appointment rows from Google Sheets for follow-up processing.
- **Configuration choices:**  
  - Reads from **Scheduled appointments**
  - No filter conditions configured
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: ⏰ Hourly Feedback Schedule Trigger
  - Output: 🔀 Filter Completed Visits
- **Version-specific requirements:**  
  Google Sheets node typeVersion 4.7.
- **Edge cases or potential failure types:**  
  - Reads all rows every hour, which may become inefficient
  - No state tracking to prevent duplicate follow-up emails
- **Sub-workflow reference:**  
  None

---

### Node: 🔀 Filter Completed Visits

- **Type and technical role:** `n8n-nodes-base.if`  
  Filters rows where the `visit` field is not empty.
- **Configuration choices:**  
  - Condition: string `notEmpty`
  - Left value: `{{$json.visit}}`
- **Key expressions or variables used:**  
  `{{$json.visit}}`
- **Input and output connections:**  
  - Input: 📋 Fetch All Appointments for Feedback
  - True output: 📧 Send Feedback Email to Patient
  - False output: unused
- **Version-specific requirements:**  
  IF node typeVersion 2.3 with condition version 3.
- **Edge cases or potential failure types:**  
  - Depends on exact column name `visit`
  - If sheet column is `Visit`, `Visited`, `Completed`, etc., no rows will pass
  - Not-empty check may send to any row with arbitrary text, not just truly completed visits
- **Sub-workflow reference:**  
  None

---

### Node: 📧 Send Feedback Email to Patient

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a plain text feedback request email.
- **Configuration choices:**  
  - Recipient: `info@example.com`
  - Subject: `Feedback Form`
  - Email type: `text`
  - Message:
    - `Hi {{$json.name}},`
    - includes `{{$json.form_link}}`
- **Key expressions or variables used:**  
  - `{{$json.name}}`
  - `{{$json.form_link}}`
- **Input and output connections:**  
  - Input: 🔀 Filter Completed Visits
  - Output: none
- **Version-specific requirements:**  
  Gmail node typeVersion 2.2.
- **Edge cases or potential failure types:**  
  - Static recipient means the email is not sent to the patient unless manually changed
  - Field-name mismatch likely: appointments sheet may contain `Patient Name`, not `name`
  - `form_link` must exist in the sheet or email body will contain an empty link
  - No deduplication or “already sent” flag
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Workflow Overview | Sticky Note | Visual documentation |  |  | ## 🏥 Healthcare Appointment Booking Agent<br>### How it works<br>This workflow automates the end-to-end healthcare appointment lifecycle across three parallel pipelines:<br>**Pipeline 1 – Patient Intake & Triage:** A webhook receives new patient lead data (name, age, gender, symptoms). The data is formatted, then passed to an AI Triage Agent (Azure OpenAI) that assesses urgency level (LOW / MEDIUM / HIGH / CRITICAL), recommends a department, and sets appointment priority. Normalized results are saved to a Google Sheet ("Patient forms" tab).<br>**Pipeline 2 – Doctor Assignment:** A Google Sheets trigger fires whenever a new patient row is added. A second AI Agent reads the doctor list from the sheet and assigns the most suitable doctor based on symptoms and specialty. The assignment is written back to the "Scheduled appointments" tab. Appointment details are then fetched and an HTML email digest is sent to the assigned doctor.<br>**Pipeline 3 – Post-Visit Feedback:** An hourly Schedule Trigger reads the appointments sheet, filters records that have a completed visit flag, and sends a personalized feedback-form email to each patient.<br>### Setup steps<br>1. **Webhook** – Activate the workflow; note the webhook URL and configure your patient intake form to POST to it.<br>2. **Azure OpenAI** – Add your Azure OpenAI credentials to both AI Agent nodes (model: `gpt-4o-mini`).<br>3. **Google Sheets** – Connect Google Sheets OAuth2 to all sheet nodes. Create a spreadsheet with three tabs: *Patient forms*, *Doctor details*, and *Scheduled appointments*.<br>4. **Gmail** – Connect Gmail OAuth2 to both email nodes and update the `sendTo` addresses.<br>5. **Schedule Trigger** – Adjust the hourly interval to match your clinic's feedback cadence.<br>### Customization<br>- Swap Azure OpenAI for any other LLM by replacing the Chat Model sub-node.<br>- Add a Slack or WhatsApp node after doctor assignment for real-time staff alerts. |
| Section: Patient Intake & Triage | Sticky Note | Visual section label |  |  | ## 📥 Patient Intake & Triage<br>Receives raw patient lead via webhook, normalizes fields, and runs AI triage to determine urgency level, recommended department, and appointment priority. Results are saved to the Patient forms sheet. |
| Section: Doctor Assignment & Notification | Sticky Note | Visual section label |  |  | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor. |
| Section: Post-Visit Feedback | Sticky Note | Visual section label |  |  | ## 📬 Post-Visit Feedback<br>Runs hourly via Schedule Trigger. Fetches appointments with a completed visit flag and sends each patient a personalized feedback form email. |
| ⚠️ Warning: Azure OpenAI Credentials | Sticky Note | Visual warning |  |  | ⚠️ **Azure OpenAI Credentials Required**<br>Both AI Agent nodes depend on this model. Ensure your Azure OpenAI API key, endpoint, and deployment name are correctly configured. Misconfiguration will silently break triage and doctor assignment. |
| ⚠️ Warning: Sheet Overwrite Risk | Sticky Note | Visual warning |  |  | ⚠️ **Google Sheets Write Risk**<br>This node appends or overwrites rows in the Scheduled Appointments sheet. Ensure column headers in the sheet exactly match the mapped fields to avoid data loss or misalignment. |
| 🔔 Patient Lead Webhook | Webhook | Receives patient intake POST requests |  | 🗂️ Format Patient Data | ## 📥 Patient Intake & Triage<br>Receives raw patient lead via webhook, normalizes fields, and runs AI triage to determine urgency level, recommended department, and appointment priority. Results are saved to the Patient forms sheet. |
| 🗂️ Format Patient Data | Set | Standardizes webhook payload fields | 🔔 Patient Lead Webhook | 🤖 AI Triage Agent | ## 📥 Patient Intake & Triage<br>Receives raw patient lead via webhook, normalizes fields, and runs AI triage to determine urgency level, recommended department, and appointment priority. Results are saved to the Patient forms sheet. |
| 🤖 AI Triage Agent | LangChain Agent | Performs AI-based patient triage | 🗂️ Format Patient Data, 🧠 Azure OpenAI – Triage Model | ⚙️ Normalize Triage Output | ## 📥 Patient Intake & Triage<br>Receives raw patient lead via webhook, normalizes fields, and runs AI triage to determine urgency level, recommended department, and appointment priority. Results are saved to the Patient forms sheet.<br>⚠️ **Azure OpenAI Credentials Required**<br>Both AI Agent nodes depend on this model. Ensure your Azure OpenAI API key, endpoint, and deployment name are correctly configured. Misconfiguration will silently break triage and doctor assignment. |
| 🧠 Azure OpenAI – Triage Model | Azure OpenAI Chat Model | Provides LLM for triage agent |  | 🤖 AI Triage Agent | ⚠️ **Azure OpenAI Credentials Required**<br>Both AI Agent nodes depend on this model. Ensure your Azure OpenAI API key, endpoint, and deployment name are correctly configured. Misconfiguration will silently break triage and doctor assignment. |
| ⚙️ Normalize Triage Output | Code | Parses and standardizes triage JSON | 🤖 AI Triage Agent | 📝 Save Triage to Patient Forms Sheet | ## 📥 Patient Intake & Triage<br>Receives raw patient lead via webhook, normalizes fields, and runs AI triage to determine urgency level, recommended department, and appointment priority. Results are saved to the Patient forms sheet. |
| 📝 Save Triage to Patient Forms Sheet | Google Sheets | Writes triage results to intake sheet | ⚙️ Normalize Triage Output |  | ## 📥 Patient Intake & Triage<br>Receives raw patient lead via webhook, normalizes fields, and runs AI triage to determine urgency level, recommended department, and appointment priority. Results are saved to the Patient forms sheet. |
| 📊 New Patient Row Trigger | Google Sheets Trigger | Detects new patient rows in sheet |  | 🤖 AI Doctor Assignment Agent | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor. |
| 🤖 AI Doctor Assignment Agent | LangChain Agent | Assigns doctor based on symptoms and sheet tool data | 📊 New Patient Row Trigger, 🧠 Azure OpenAI – Assignment Model, 📋 Fetch Doctor List from Sheet | ⚙️ Normalize Assignment Output | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor.<br>⚠️ **Azure OpenAI Credentials Required**<br>Both AI Agent nodes depend on this model. Ensure your Azure OpenAI API key, endpoint, and deployment name are correctly configured. Misconfiguration will silently break triage and doctor assignment. |
| 🧠 Azure OpenAI – Assignment Model | Azure OpenAI Chat Model | Provides LLM for assignment agent |  | 🤖 AI Doctor Assignment Agent | ⚠️ **Azure OpenAI Credentials Required**<br>Both AI Agent nodes depend on this model. Ensure your Azure OpenAI API key, endpoint, and deployment name are correctly configured. Misconfiguration will silently break triage and doctor assignment. |
| 📋 Fetch Doctor List from Sheet | Google Sheets Tool | Exposes doctor list as AI tool |  | 🤖 AI Doctor Assignment Agent | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor. |
| ⚙️ Normalize Assignment Output | Code | Parses and standardizes assignment JSON | 🤖 AI Doctor Assignment Agent | 📝 Save Assignment to Appointments Sheet | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor. |
| 📝 Save Assignment to Appointments Sheet | Google Sheets | Writes assignments to schedule sheet | ⚙️ Normalize Assignment Output | 📋 Fetch Appointments for Doctor Email | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor.<br>⚠️ **Google Sheets Write Risk**<br>This node appends or overwrites rows in the Scheduled Appointments sheet. Ensure column headers in the sheet exactly match the mapped fields to avoid data loss or misalignment. |
| 📋 Fetch Appointments for Doctor Email | Google Sheets | Reads appointments for doctor email digest | 📝 Save Assignment to Appointments Sheet | ⚙️ Build Doctor Schedule Email HTML | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor. |
| ⚙️ Build Doctor Schedule Email HTML | Code | Builds HTML schedule email | 📋 Fetch Appointments for Doctor Email | 📧 Send Schedule Email to Doctor | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor. |
| 📧 Send Schedule Email to Doctor | Gmail | Sends doctor schedule email | ⚙️ Build Doctor Schedule Email HTML |  | ## 🩺 Doctor Assignment<br>Triggered when a new patient row lands in Google Sheets. An AI Agent consults the doctor list and assigns the best-matched specialist. The appointment is written back to the sheet and a schedule digest email is sent to the doctor. |
| ⏰ Hourly Feedback Schedule Trigger | Schedule Trigger | Starts hourly feedback scan |  | 📋 Fetch All Appointments for Feedback | ## 📬 Post-Visit Feedback<br>Runs hourly via Schedule Trigger. Fetches appointments with a completed visit flag and sends each patient a personalized feedback form email. |
| 📋 Fetch All Appointments for Feedback | Google Sheets | Reads appointments for feedback | ⏰ Hourly Feedback Schedule Trigger | 🔀 Filter Completed Visits | ## 📬 Post-Visit Feedback<br>Runs hourly via Schedule Trigger. Fetches appointments with a completed visit flag and sends each patient a personalized feedback form email. |
| 🔀 Filter Completed Visits | IF | Keeps only rows with visit flag | 📋 Fetch All Appointments for Feedback | 📧 Send Feedback Email to Patient | ## 📬 Post-Visit Feedback<br>Runs hourly via Schedule Trigger. Fetches appointments with a completed visit flag and sends each patient a personalized feedback form email. |
| 📧 Send Feedback Email to Patient | Gmail | Sends feedback request email | 🔀 Filter Completed Visits |  | ## 📬 Post-Visit Feedback<br>Runs hourly via Schedule Trigger. Fetches appointments with a completed visit flag and sends each patient a personalized feedback form email. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Healthcare - Appointment Booking Agent`.

2. **Add a Sticky Note for overall description**
   - Add a sticky note titled `📋 Workflow Overview`.
   - Paste a high-level explanation of the 3 pipelines if desired.

3. **Add the patient intake webhook**
   - Create a **Webhook** node named `🔔 Patient Lead Webhook`.
   - Set:
     - HTTP Method: `POST`
     - Path: `patient-lead`
   - Leave other options default.

4. **Add the patient formatting node**
   - Create a **Set** node named `🗂️ Format Patient Data`.
   - Add string fields:
     - `patient_name` = `{{$json["name"]}}`
     - `phone` = `{{$json["phone"]}}`
     - `symptoms` = `{{$json["symptoms"]}}`
     - `age` = `{{$json["age"]}}`
     - `gender` = `{{$json["gender"]}}`
     - `additonal notes` = `{{$json["additonal notes"]}}`
   - Connect `🔔 Patient Lead Webhook` → `🗂️ Format Patient Data`.

5. **Add the triage AI agent**
   - Create an **AI Agent** node named `🤖 AI Triage Agent`.
   - Set prompt mode to manually define the prompt.
   - In the user prompt, include patient fields:
     - patient name
     - age
     - gender
     - phone
     - symptoms
     - additional notes
   - In the system message, instruct the model to:
     - avoid diagnosis
     - classify urgency
     - recommend department
     - output JSON only
     - use LOW / MEDIUM / HIGH / CRITICAL
   - Connect `🗂️ Format Patient Data` → `🤖 AI Triage Agent`.

6. **Add the Azure OpenAI model for triage**
   - Create an **Azure OpenAI Chat Model** node named `🧠 Azure OpenAI – Triage Model`.
   - Select or create Azure OpenAI credentials:
     - Azure endpoint
     - API key
     - deployment/model access
   - Set model to `gpt-4o-mini`.
   - Connect the model node to the AI agent via the **AI language model** connection.

7. **Add the triage normalization code**
   - Create a **Code** node named `⚙️ Normalize Triage Output`.
   - Paste logic that:
     - parses string or JSON AI output
     - normalizes urgency
     - maps urgency to appointment priority
     - creates a `sheet_row` object
   - Connect `🤖 AI Triage Agent` → `⚙️ Normalize Triage Output`.

8. **Create the Google Sheets storage for triage**
   - Prepare a Google spreadsheet with tabs:
     - `Patient forms`
     - `Doctor details`
     - `Scheduled appointments`
   - In `Patient forms`, create headers matching intended fields, ideally:
     - `Name`
     - `Age`
     - `Gender`
     - `Phone`
     - `Symptoms`
     - `Urgency`
     - `Department`
     - `Priority`
     - `Recommended Action`

9. **Add the triage save node**
   - Create a **Google Sheets** node named `📝 Save Triage to Patient Forms Sheet`.
   - Configure Google Sheets OAuth2 credentials.
   - Choose the spreadsheet.
   - Select tab `Patient forms`.
   - Set operation to `appendOrUpdate`.
   - Use auto-map input data.
   - Connect `⚙️ Normalize Triage Output` → `📝 Save Triage to Patient Forms Sheet`.

10. **Add the doctor assignment trigger**
    - Create a **Google Sheets Trigger** node named `📊 New Patient Row Trigger`.
    - Configure Google Sheets Trigger OAuth2 credentials.
    - Select the same spreadsheet and the `Patient forms` sheet.
    - Set polling to every minute.

11. **Add the doctor assignment AI agent**
    - Create an **AI Agent** node named `🤖 AI Doctor Assignment Agent`.
    - Build a prompt that includes:
      - patient name
      - age
      - gender
      - symptoms
    - In the system message, instruct the model to:
      - always consult the doctor list tool first
      - choose only a doctor from the sheet
      - map symptoms to specialty
      - fall back to General Physician
      - return JSON only
    - Connect `📊 New Patient Row Trigger` → `🤖 AI Doctor Assignment Agent`.

12. **Add the Azure OpenAI model for doctor assignment**
    - Create another **Azure OpenAI Chat Model** node named `🧠 Azure OpenAI – Assignment Model`.
    - Use the same Azure OpenAI credentials.
    - Set model to `gpt-4o-mini`.
    - Connect it to `🤖 AI Doctor Assignment Agent` via the **AI language model** port.

13. **Prepare the doctor details sheet**
    - In the `Doctor details` tab, add doctor records with at least:
      - doctor name
      - specialty
      - optionally availability, email, shift
    - Keep specialty names consistent.

14. **Add the doctor list tool node**
    - Create a **Google Sheets Tool** node named `📋 Fetch Doctor List from Sheet`.
    - Select the same spreadsheet.
    - Choose sheet `Doctor details`.
    - Connect it to `🤖 AI Doctor Assignment Agent` using the **AI tool** connection.

15. **Add the assignment normalization code**
    - Create a **Code** node named `⚙️ Normalize Assignment Output`.
    - Paste logic that:
      - extracts JSON from text if needed
      - normalizes gender and age
      - normalizes specialty labels
      - builds a final `sheet_row`
   - Connect `🤖 AI Doctor Assignment Agent` → `⚙️ Normalize Assignment Output`.

16. **Prepare the appointments sheet**
    - In the `Scheduled appointments` tab, add headers for assignment and later follow-up, such as:
      - `Patient Name`
      - `Age`
      - `Gender`
      - `Symptoms`
      - `Specialty`
      - `Doctor Assigned`
      - `visit`
      - `form_link`
      - `time`
      - `doctor_name`
      - `name`
   - For this exact workflow, you may need duplicate or compatibility columns because downstream nodes expect inconsistent field names.

17. **Add the assignment save node**
    - Create a **Google Sheets** node named `📝 Save Assignment to Appointments Sheet`.
    - Set operation to `appendOrUpdate`.
    - Select `Scheduled appointments`.
    - Enable auto-map input data.
    - Connect `⚙️ Normalize Assignment Output` → `📝 Save Assignment to Appointments Sheet`.

18. **Add the node to fetch appointments for the doctor email**
    - Create a **Google Sheets** node named `📋 Fetch Appointments for Doctor Email`.
    - Configure it to read from `Scheduled appointments`.
    - Leave filters empty if reproducing exactly.
    - Connect `📝 Save Assignment to Appointments Sheet` → `📋 Fetch Appointments for Doctor Email`.

19. **Add the HTML email builder**
    - Create a **Code** node named `⚙️ Build Doctor Schedule Email HTML`.
    - Add code that:
      - reads all incoming rows with `$input.all()`
      - builds an HTML table
      - returns `{ email_html: html }`
    - Connect `📋 Fetch Appointments for Doctor Email` → `⚙️ Build Doctor Schedule Email HTML`.

20. **Add the Gmail node for doctor notifications**
    - Create a **Gmail** node named `📧 Send Schedule Email to Doctor`.
    - Configure Gmail OAuth2 credentials.
    - Set:
      - To: `info@example.com` or replace with a real doctor email
      - Subject: `Today's Appointment Schedule`
      - Message: `{{$json.email_html}}`
    - Connect `⚙️ Build Doctor Schedule Email HTML` → `📧 Send Schedule Email to Doctor`.

21. **Add the hourly scheduler**
    - Create a **Schedule Trigger** node named `⏰ Hourly Feedback Schedule Trigger`.
    - Set it to run every 1 hour.

22. **Add appointment retrieval for feedback**
    - Create a **Google Sheets** node named `📋 Fetch All Appointments for Feedback`.
    - Configure it to read from `Scheduled appointments`.
    - Connect `⏰ Hourly Feedback Schedule Trigger` → `📋 Fetch All Appointments for Feedback`.

23. **Add the completion filter**
    - Create an **IF** node named `🔀 Filter Completed Visits`.
    - Configure one condition:
      - left value: `{{$json.visit}}`
      - operator: `not empty`
    - Connect `📋 Fetch All Appointments for Feedback` → `🔀 Filter Completed Visits`.

24. **Add the Gmail node for patient feedback**
    - Create a **Gmail** node named `📧 Send Feedback Email to Patient`.
    - Configure Gmail OAuth2 credentials.
    - Set:
      - To: `info@example.com` if reproducing exactly, or replace with patient email field
      - Subject: `Feedback Form`
      - Email type: `text`
      - Message:
        - `Hi {{$json.name}},`
        - `Please rate your experience!`
        - `here is the feedback form: {{$json.form_link}}`
    - Connect the **true** output of `🔀 Filter Completed Visits` → `📧 Send Feedback Email to Patient`.

25. **Add optional sticky notes**
    - Add section notes for:
      - patient intake
      - doctor assignment
      - post-visit feedback
    - Add warning notes for:
      - Azure OpenAI credentials
      - Google Sheets overwrite risk

26. **Review field naming carefully**
    - To reproduce the workflow exactly, keep current expressions.
    - To make it function reliably, align field names across:
      - webhook payload
      - AI prompts
      - normalization code
      - Google Sheets headers
      - email templates

27. **Activate credentials**
    - Azure OpenAI credentials for both model nodes
    - Google Sheets OAuth2 for all Google Sheets nodes
    - Google Sheets Trigger OAuth2 for the trigger node
    - Gmail OAuth2 for both Gmail nodes

28. **Test each entry point**
    - Test webhook with a sample POST body
    - Add a row manually to `Patient forms` to verify doctor assignment
    - Run the schedule branch manually to verify feedback email behavior

29. **Activate the workflow**
    - Required for:
      - production webhook endpoint
      - Google Sheets trigger polling
      - hourly schedule trigger

### Sub-workflow setup
This workflow does **not** use Execute Workflow nodes or child workflows.  
There are no sub-workflow parameters or callback contracts to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow contains three independent entry points: webhook, Google Sheets trigger, and schedule trigger. | Architecture note |
| Both AI pipelines depend on Azure OpenAI model availability and correct deployment configuration. | Azure OpenAI setup |
| The spreadsheet is expected to contain three tabs: Patient forms, Doctor details, and Scheduled appointments. | Google Sheets setup |
| Several field-name mismatches exist in the current design, including `additonal notes` vs `notes`, and mixed casing such as `Name` vs `name`. These should be aligned before production use. | Reliability note |
| The doctor notification email currently uses a static recipient and unfiltered appointment fetch. | Operational limitation |
| The feedback email currently uses a static recipient and no sent-status tracking, so duplicate sends are possible. | Operational limitation |