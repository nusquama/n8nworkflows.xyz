Register warranties and send GPT-4o-mini reminders with GoHighLevel and Sheets

https://n8nworkflows.xyz/workflows/register-warranties-and-send-gpt-4o-mini-reminders-with-gohighlevel-and-sheets-20434


# Register warranties and send GPT-4o-mini reminders with GoHighLevel and Sheets

### 1. Workflow Overview

This workflow is designed for appliance and automobile dealers to automate product warranty registrations and manage lifecycle service/warranty reminders via GoHighLevel (GHL) and Google Sheets. It supports two independent entry paths: a hosted n8n Form submission and a web-based POS/DMS integration webhook.

The workflow is logically structured into the following operational blocks:
- **1.1 Registration Intake & Normalization:** Receives input from either the n8n form or a webhook, merges global configuration variables, and cleans/normalizes payload fields (e.g., phone numbers, purchase dates, category-based warranty rules, and service schedules).
- **1.2 Duplicate Detection & CRM Upsert:** Fetches current warranty records from Google Sheets to screen out duplicate serial/VIN numbers, then upserts the buyer as a contact inside GoHighLevel based on their phone number.
- **1.3 Registry Storage & Welcome Processing:** Computes warranty expiration dates and service intervals, appends the new record to Google Sheets, tags and notes the contact in GoHighLevel, and delivers an initial welcome SMS/message alongside an HTML warranty certificate via email.
- **1.4 Daily Reminder Scheduling & Evaluation:** Operates on a daily schedule (9:30 AM), reads all active entries from Google Sheets, and applies chronological offset calculations to detect upcoming services, overdue tasks, or expiring warranties.
- **1.5 AI Copy Generation & Multi-Channel Dispatch:** Feeds qualifying reminder milestones to OpenAI (`gpt-4o-mini`) to generate tailored messaging content, compiling multi-channel notifications dispatched through GoHighLevel conversations.
- **1.6 Stage Tracking, Lifecycle Updates, & Opportunity Creation:** Updates Google Sheets reminder status markers to prevent duplicate notifications, and automatically opens pipeline opportunities in GoHighLevel when a new service or renewal cycle begins.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Registration Intake & Normalization
- **Overview:** Captures incoming warranty details from customers or external software, applies global configuration settings, and standardizes data formats using programmatic rules.
- **Nodes Involved:** 
  - `Warranty Registration Form`
  - `Receive Sale from DMS / POS`
  - `Set Registration Config`
  - `Normalize Registration Data`

- **Node Details:**
  - **Warranty Registration Form**
    - *Type & Technical Role:* `n8n-nodes-base.formTrigger` (Trigger Node)
    - *Configuration:* Hosted web form configured with mandatory input fields (Full Name, Mobile Number, Product Category, Brand, Model, Serial/VIN, Invoice Number, Purchase Date, and Consent checkbox) and custom success message text.
    - *Input/Output:* Output connects to `Set Registration Config`.
    - *Edge Cases:* Missing mandatory form fields prevent trigger execution; invalid mobile formats handled downstream.
  - **Receive Sale from DMS / POS**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Trigger Node)
    - *Configuration:* Listens on path `warranty-registration` for incoming `POST` requests.
    - *Input/Output:* Output connects to `Set Registration Config`.
    - *Edge Cases:* Unauthenticated payloads or malformed JSON formats.
  - **Set Registration Config**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Assigns global constants including dealership branding details, calendar booking URLs, Google Review links, GoHighLevel API base URLs, location IDs, pipeline identifiers, Google Sheet IDs, and offset intervals.
    - *Input/Output:* Inputs from both triggers; output connects to `Normalize Registration Data`.
  - **Normalize Registration Data**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Normalizes data attributes, handles payload extraction from both webhook bodies and form submissions, maps product categories to predefined rule sets (warranty duration, service intervals, free service quotas), calculates purchase dates, and offsets late registrations.
    - *Key Expressions:* Accesses `$input.all()`, processes input properties dynamically via fallback extraction helpers.
    - *Input/Output:* Input from `Set Registration Config`; output connects to `Get Existing Registrations`.
    - *Edge Cases:* Missing customer name or serial number causes records to be dropped.

---

#### Block 1.2: Duplicate Detection & CRM Contact
- **Overview:** Prevents duplicate warranty creations by cross-referencing serial numbers against existing registry sheets and upserting valid buyers into GoHighLevel.
- **Nodes Involved:**
  - `Get Existing Registrations`
  - `Skip Duplicate Serials`
  - `Upsert Contact in GHL`

- **Node Details:**
  - **Get Existing Registrations**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Integration)
    - *Configuration:* Reads rows from the configured Google Sheets spreadsheet and registry tab. Executed once (`executeOnce: true`).
    - *Input/Output:* Input from `Normalize Registration Data`; output connects to `Skip Duplicate Serials`.
    - *Edge Cases:* Authentication failures with Google OAuth2 or missing spreadsheet tabs.
  - **Skip Duplicate Serials**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Compares incoming normalized serial/VIN numbers against cached Google Sheet entries using a JavaScript `Set` to drop duplicates.
    - *Input/Output:* Inputs from `Normalize Registration Data` and `Get Existing Registrations`; output connects to `Upsert Contact in GHL`.
  - **Upsert Contact in GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Sends a `POST` request to `{ghl_api_base}/contacts/upsert` with location ID, name, email, and phone parameters using Generic Credential Type (HTTP Header Auth). Retries up to 3 times on failure.
    - *Input/Output:* Input from `Skip Duplicate Serials`; output connects to `Build Registration Record & Certificate`.
    - *Edge Cases:* Invalid API bearer token, rate limits, or network timeouts.

---

#### Block 1.3: Registry Storage & Welcome Processing
- **Overview:** Builds formatted warranty records and HTML certificates, appends data to the Google Sheets registry, tags/notes the contact in GoHighLevel, and dispatches initial welcome messages.
- **Nodes Involved:**
  - `Build Registration Record & Certificate`
  - `Add to Warranty Registry Sheet`
  - `Tag Contact in GHL`
  - `Add Registration Note in GHL`
  - `Send Welcome Message via GHL`
  - `Email Warranty Certificate via GHL`

- **Node Details:**
  - **Build Registration Record & Certificate**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Compiles row objects matching schema column definitions, generates styled HTML warranty certificates, and drafts initial SMS text and metadata tags/notes.
    - *Input/Output:* Input from `Upsert Contact in GHL`; output connects to `Add to Warranty Registry Sheet`.
  - **Add to Warranty Registry Sheet**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Integration)
    - *Configuration:* Appends row items matching configured column schemas into the Google Sheets registry.
    - *Input/Output:* Input from `Build Registration Record & Certificate`; outputs connect in parallel to `Tag Contact in GHL`, `Add Registration Note in GHL`, `Send Welcome Message via GHL`, and `Email Warranty Certificate via GHL`.
  - **Tag Contact in GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Sends a `POST` request to `{ghl_api_base}/contacts/{contact_id}/tags` to apply metadata tags (e.g., category slug, brand slug, warranty type).
    - *Input/Output:* Input from `Add to Warranty Registry Sheet`.
  - **Add Registration Note in GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Posts registration audit notes to `{ghl_api_base}/contacts/{contact_id}/notes`.
    - *Input/Output:* Input from `Add to Warranty Registry Sheet`.
  - **Send Welcome Message via GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Posts welcome messages via configured communication channel to `{ghl_api_base}/conversations/messages`.
    - *Input/Output:* Input from `Add to Warranty Registry Sheet`.
  - **Email Warranty Certificate via GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Dispatches HTML certificates via email to `{ghl_api_base}/conversations/messages`.
    - *Input/Output:* Input from `Add to Warranty Registry Sheet`.

---

#### Block 1.4: Daily Reminder Scheduling & Evaluation
- **Overview:** Executes a daily cron check at 9:30 AM, reads the entire warranty registry, and calculates milestone offsets to identify due services, overdue events, or expiring warranties.
- **Nodes Involved:**
  - `Daily 9:30 AM Schedule`
  - `Set Reminder Config`
  - `Get Warranty Registry`
  - `Plan Due Reminders`

- **Node Details:**
  - **Daily 9:30 AM Schedule**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration:* Cron expression trigger set to run daily at `30 9 * * *`.
    - *Input/Output:* Output connects to `Set Reminder Config`.
  - **Set Reminder Config**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Assigns global reminder configurations, offset windows, pipeline stages, and integration parameters.
    - *Input/Output:* Input from `Daily 9:30 AM Schedule`; output connects to `Get Warranty Registry`.
  - **Get Warranty Registry**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Integration)
    - *Configuration:* Reads all data rows from the `Warranty Registry` tab. Executed once (`executeOnce: true`).
    - *Input/Output:* Input from `Set Reminder Config`; output connects to `Plan Due Reminders`.
  - **Plan Due Reminders**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Evaluates active records against service/warranty offset schedules, preventing duplicate notifications by checking stage markers.
    - *Input/Output:* Input from `Get Warranty Registry`; output connects to `AI Write Reminder Copy`.

---

#### Block 1.5: AI Copy Generation & Multi-Channel Dispatch
- **Overview:** Uses OpenAI (`gpt-4o-mini`) to generate customized SMS and email reminder text, which is parsed and sent through GoHighLevel.
- **Nodes Involved:**
  - `AI Write Reminder Copy`
  - `Compose Reminder Messages`
  - `Send Reminder Message via GHL`
  - `Send Reminder Email via GHL`
  - `Tag Reminder Stage in GHL`
  - `Log Reminder Note in GHL`

- **Node Details:**
  - **AI Write Reminder Copy**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Model Node)
    - *Configuration:* Uses model `gpt-4o-mini` with a temperature of `0.5` and JSON output mode configured with structured system prompts.
    - *Input/Output:* Input from `Plan Due Reminders`; output connects to `Compose Reminder Messages`.
    - *Edge Cases:* API errors or JSON parsing issues handled via fallbacks in the downstream code node.
  - **Compose Reminder Messages**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Parses AI outputs, applies robust fallback messaging structures, generates HTML email wrappers, and constructs payload structures for GHL dispatch.
    - *Input/Output:* Input from `AI Write Reminder Copy`; outputs connect in parallel to downstream communication and update nodes.
  - **Send Reminder Message via GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Sends the reminder SMS via `{ghl_api_base}/conversations/messages`.
    - *Input/Output:* Input from `Compose Reminder Messages`.
  - **Send Reminder Email via GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Sends reminder emails via `{ghl_api_base}/conversations/messages`.
    - *Input/Output:* Input from `Compose Reminder Messages`.
  - **Tag Reminder Stage in GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Applies reminder stage tags to contact profiles via `{ghl_api_base}/contacts/{contact_id}/tags`.
    - *Input/Output:* Input from `Compose Reminder Messages`.
  - **Log Reminder Note in GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Logs audit notes of sent reminders to `{ghl_api_base}/contacts/{contact_id}/notes`.
    - *Input/Output:* Input from `Compose Reminder Messages`.

---

#### Block 1.6: Stage Tracking, Lifecycle Updates, & Opportunity Creation
- **Overview:** Updates the Google Sheets registry with reminder tracking stages and manages new business opportunity cycles within GoHighLevel.
- **Nodes Involved:**
  - `Prepare Stage Update`
  - `Update Reminder Stage in Sheet`
  - `New Revenue Cycle?`
  - `Create Opportunity in GHL`
  - `Prepare Opportunity Link`
  - `Save Opportunity ID to Sheet`

- **Node Details:**
  - **Prepare Stage Update**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Extracts tracker object schemas for sheet updates.
    - *Input/Output:* Input from `Compose Reminder Messages`; output connects to `Update Reminder Stage in Sheet`.
  - **Update Reminder Stage in Sheet**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Integration)
    - *Configuration:* Updates existing rows in Google Sheets matching `registration_id`.
    - *Input/Output:* Input from `Prepare Stage Update`.
  - **New Revenue Cycle?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration:* Evaluates whether `create_opportunity` evaluates to true.
    - *Input/Output:* Input from `Compose Reminder Messages`; output routes to `Create Opportunity in GHL`.
  - **Create Opportunity in GHL**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Creates pipeline opportunities via `{ghl_api_base}/opportunities/`.
    - *Input/Output:* Input from `New Revenue Cycle?`; output connects to `Prepare Opportunity Link`.
  - **Prepare Opportunity Link**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Maps created opportunity IDs to correct spreadsheet mapping columns (service or warranty opportunity ID fields).
    - *Input/Output:* Input from `Create Opportunity in GHL`; output connects to `Save Opportunity ID to Sheet`.
  - **Save Opportunity ID to Sheet**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Integration)
    - *Configuration:* Updates target registry rows with newly generated opportunity IDs.
    - *Input/Output:* Input from `Prepare Opportunity Link`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🛠️ Warranty Registration & Service Reminders (HighLevel)... |
| Sticky Note – Registration Intake | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📝 Registration Intake... |
| Sticky Note – Dedupe & CRM Contact | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🔍 Dedupe & CRM Contact... |
| Sticky Note – Save to Registry | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🗂️ Save to Registry... |
| Sticky Note – Welcome in GHL | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📨 Welcome in GHL... |
| Sticky Note – Daily Reminder Planning | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📅 Daily Reminder Planning... |
| Sticky Note – AI Reminder Copy | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🧠 AI Reminder Copy... |
| Sticky Note – Send & Log in GHL | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📣 Send & Log in GHL... |
| Sticky Note – Stage Tracking & Opportunities | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 💰 Stage Tracking & Opportunities... |
| Sticky Note – Credentials & Security | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🔐 Credentials & Security... |
| Warranty Registration Form | `n8n-nodes-base.formTrigger` | Capture user form submissions | None | Set Registration Config | ## 📝 Registration Intake |
| Receive Sale from DMS / POS | `n8n-nodes-base.webhook` | Webhook intake for external sales | None | Set Registration Config | ## 📝 Registration Intake |
| Set Registration Config | `n8n-nodes-base.set` | Define global configuration variables | Warranty Registration Form, Receive Sale from DMS / POS | Normalize Registration Data | ## 📝 Registration Intake |
| Normalize Registration Data | `n8n-nodes-base.code` | Clean and normalize input parameters | Set Registration Config | Get Existing Registrations | ## 📝 Registration Intake |
| Get Existing Registrations | `n8n-nodes-base.googleSheets` | Read existing sheet rows for deduplication | Normalize Registration Data | Skip Duplicate Serials | ## 🔍 Dedupe & CRM Contact |
| Skip Duplicate Serials | `n8n-nodes-base.code` | Filter out duplicate serial/VIN inputs | Get Existing Registrations | Upsert Contact in GHL | ## 🔍 Dedupe & CRM Contact |
| Upsert Contact in GHL | `n8n-nodes-base.httpRequest` | Upsert buyer contact profile in CRM | Skip Duplicate Serials | Build Registration Record & Certificate | ## 🔍 Dedupe & CRM Contact |
| Build Registration Record & Certificate | `n8n-nodes-base.code` | Build schema records and HTML certificates | Upsert Contact in GHL | Add to Warranty Registry Sheet | ## 🗂️ Save to Registry |
| Add to Warranty Registry Sheet | `n8n-nodes-base.googleSheets` | Append new warranty row to spreadsheet | Build Registration Record & Certificate | Tag Contact in GHL, Add Registration Note in GHL, Send Welcome Message via GHL, Email Warranty Certificate via GHL | ## 🗂️ Save to Registry |
| Tag Contact in GHL | `n8n-nodes-base.httpRequest` | Apply categorical tags to contact profile | Add to Warranty Registry Sheet | None | ## 📨 Welcome in GHL |
| Add Registration Note in GHL | `n8n-nodes-base.httpRequest` | Log warranty details as a contact note | Add to Warranty Registry Sheet | None | ## 📨 Welcome in GHL |
| Send Welcome Message via GHL | `n8n-nodes-base.httpRequest` | Dispatch welcome SMS via conversations | Add to Warranty Registry Sheet | None | ## 📨 Welcome in GHL |
| Email Warranty Certificate via GHL | `n8n-nodes-base.httpRequest` | Dispatch warranty certificate email | Add to Warranty Registry Sheet | None | ## 📨 Welcome in GHL |
| Daily 9:30 AM Schedule | `n8n-nodes-base.scheduleTrigger` | Daily cron trigger for reminder processing | None | Set Reminder Config | ## 📅 Daily Reminder Planning |
| Set Reminder Config | `n8n-nodes-base.set` | Set global configurations for reminders | Daily 9:30 AM Schedule | Get Warranty Registry | ## 📅 Daily Reminder Planning |
| Get Warranty Registry | `n8n-nodes-base.googleSheets` | Fetch active warranty register rows | Set Reminder Config | Plan Due Reminders | ## 📅 Daily Reminder Planning |
| Plan Due Reminders | `n8n-nodes-base.code` | Evaluate milestone offsets and stages | Get Warranty Registry | AI Write Reminder Copy | ## 📅 Daily Reminder Planning |
| AI Write Reminder Copy | `@n8n/n8n-nodes-langchain.openAi` | Draft reminder messages using GPT-4o-mini | Plan Due Reminders | Compose Reminder Messages | ## 🧠 AI Reminder Copy |
| Compose Reminder Messages | `n8n-nodes-base.code` | Parse AI output and build dispatch payloads | AI Write Reminder Copy | Send Reminder Message via GHL, Send Reminder Email via GHL, Tag Reminder Stage in GHL, Log Reminder Note in GHL, Prepare Stage Update, New Revenue Cycle? | ## 🧠 AI Reminder Copy |
| Send Reminder Message via GHL | `n8n-nodes-base.httpRequest` | Send reminder SMS via GHL conversations | Compose Reminder Messages | None | ## 📣 Send & Log in GHL |
| Send Reminder Email via GHL | `n8n-nodes-base.httpRequest` | Send reminder email via GHL conversations | Compose Reminder Messages | None | ## 📣 Send & Log in GHL |
| Tag Reminder Stage in GHL | `n8n-nodes-base.httpRequest` | Apply reminder milestone tags in GHL | Compose Reminder Messages | None | ## 📣 Send & Log in GHL |
| Log Reminder Note in GHL | `n8n-nodes-base.httpRequest` | Log reminder communication notes | Compose Reminder Messages | None | ## 📣 Send & Log in GHL |
| Prepare Stage Update | `n8n-nodes-base.code` | Format tracking data for sheet update | Compose Reminder Messages | Update Reminder Stage in Sheet | ## 💰 Stage Tracking & Opportunities |
| New Revenue Cycle? | `n8n-nodes-base.if` | Check if new opportunity should be created | Compose Reminder Messages | Create Opportunity in GHL | ## 💰 Stage Tracking & Opportunities |
| Update Reminder Stage in Sheet | `n8n-nodes-base.googleSheets` | Update milestone reminder stages in sheet | Prepare Stage Update | None | ## 💰 Stage Tracking & Opportunities |
| Create Opportunity in GHL | `n8n-nodes-base.httpRequest` | Create new business opportunity in CRM | New Revenue Cycle? | Prepare Opportunity Link | ## 💰 Stage Tracking & Opportunities |
| Prepare Opportunity Link | `n8n-nodes-base.code` | Map opportunity ID to correct column schema | Create Opportunity in GHL | Save Opportunity ID to Sheet | ## 💰 Stage Tracking & Opportunities |
| Save Opportunity ID to Sheet | `n8n-nodes-base.googleSheets` | Save generated opportunity ID to spreadsheet | Prepare Opportunity Link | None | ## 💰 Stage Tracking & Opportunities |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually inside n8n:

1. **Create Form Trigger Node:**
   - Add a **Form Trigger** (`n8n-nodes-base.formTrigger`) named `Warranty Registration Form`.
   - Set path to `warranty-registration-form`. Configure form title (`Register Your Warranty`) and add required fields: Full Name, Mobile Number, Product Category (Dropdown: Car, Two Wheeler, Air Conditioner, etc.), Brand, Model, Serial / VIN / Chassis Number, Invoice Number, Purchase Date (Date), Extended Warranty (Dropdown), Current Odometer (Number), and Consent (Checkbox).
2. **Create Webhook Trigger Node:**
   - Add a **Webhook** (`n8n-nodes-base.webhook`) named `Receive Sale from DMS / POS`.
   - Set HTTP Method to `POST` and path to `warranty-registration`.
3. **Create Registration Configuration Node:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Set Registration Config`.
   - Assign configuration properties including `dealer_name`, `dealer_phone`, `dealer_address`, `booking_link`, `review_link`, `ghl_api_base`, `ghl_api_version`, `ghl_location_id`, `msg_channel`, `ghl_pipeline_id`, `sheet_id`, `registry_sheet` (`Warranty Registry`), and offset variables. Connect both triggers to this node.
4. **Create Normalization Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Normalize Registration Data`.
   - Insert JavaScript logic to extract payload fields, clean strings, handle date conversions, calculate category rules, and determine initial service schedules. Connect `Set Registration Config` output here.
5. **Create Google Sheets Lookup Node:**
   - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Get Existing Registrations`.
   - Configure operation to read rows, link using Google Sheets OAuth2 credentials, and reference sheet configuration expressions. Enable `Execute Once`. Connect `Normalize Registration Data` output here.
6. **Create Deduplication Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Skip Duplicate Serials`.
   - Insert code filtering out input serial numbers already present in the existing spreadsheet. Connect `Get Existing Registrations` output here.
7. **Create GHL Contact Upsert Node:**
   - Add an **HTTP Request** (`n8n-nodes-base.httpRequest`) named `Upsert Contact in GHL`.
   - Configure method `POST`, URL `={{ $('Set Registration Config').first().json.ghl_api_base }}/contacts/upsert`, and set Authentication to Generic Credential Type (HTTP Header Auth). Pass required parameters in JSON body. Connect `Skip Duplicate Serials` output here.
8. **Create Registration Record Builder Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Build Registration Record & Certificate`.
   - Insert code generating row data structures, service tables, and HTML email templates. Connect `Upsert Contact in GHL` output here.
9. **Create Google Sheets Appender Node:**
   - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Add to Warranty Registry Sheet`.
   - Configure operation to append rows using mapped schema fields from previous node outputs. Connect `Build Registration Record & Certificate` output here.
10. **Create Welcome Dispatch & Logging Nodes:**
    - Add four **HTTP Request** (`n8n-nodes-base.httpRequest`) nodes connected in parallel to `Add to Warranty Registry Sheet`:
      - `Tag Contact in GHL`: POST request to `/contacts/{contact_id}/tags`.
      - `Add Registration Note in GHL`: POST request to `/contacts/{contact_id}/notes`.
      - `Send Welcome Message via GHL`: POST request to `/conversations/messages` for SMS channels.
      - `Email Warranty Certificate via GHL`: POST request to `/conversations/messages` for email channels.
11. **Create Daily Schedule Trigger & Reminder Config:**
    - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) named `Daily 9:30 AM Schedule` set to cron `30 9 * * *`.
    - Add a **Set** node (`n8n-nodes-base.set`) named `Set Reminder Config` mirroring registration configuration properties. Connect schedule trigger to this node.
12. **Create Reminder Planning Nodes:**
    - Add a **Google Sheets** node named `Get Warranty Registry` to load all registry rows (`Execute Once`). Connect `Set Reminder Config` output here.
    - Add a **Code** node named `Plan Due Reminders` to calculate due milestone dates and offsets. Connect registry output here.
13. **Create AI Reminder Processing Nodes:**
    - Add an **OpenAI** node (`@n8n/n8n-nodes-langchain.openAi`) named `AI Write Reminder Copy`. Select model `gpt-4o-mini`, set temperature to `0.5`, enable JSON output, and provide system/user prompt instructions. Connect `Plan Due Reminders` output here.
    - Add a **Code** node named `Compose Reminder Messages` to parse AI responses, apply fallbacks, and construct outbound message payloads. Connect OpenAI output here.
14. **Create Reminder Dispatch Nodes:**
    - Add four **HTTP Request** nodes connected in parallel to `Compose Reminder Messages`:
      - `Send Reminder Message via GHL`: POST to `/conversations/messages`.
      - `Send Reminder Email via GHL`: POST to `/conversations/messages`.
      - `Tag Reminder Stage in GHL`: POST to `/contacts/{contact_id}/tags`.
      - `Log Reminder Note in GHL`: POST to `/contacts/{contact_id}/notes`.
15. **Create Stage Tracking & Opportunity Management Nodes:**
    - Add a **Code** node named `Prepare Stage Update` and a **Google Sheets** node named `Update Reminder Stage in Sheet` (Update operation matching `registration_id`). Connect from `Compose Reminder Messages`.
    - Add an **If** node named `New Revenue Cycle?` evaluating `create_opportunity`. Connect from `Compose Reminder Messages`.
    - Add an **HTTP Request** node named `Create Opportunity in GHL` (POST to `/opportunities/`), a **Code** node named `Prepare Opportunity Link`, and a **Google Sheets** node named `Save Opportunity ID to Sheet` (Update operation) to complete the opportunity lifecycle loop.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| GoHighLevel Private Integration Token Setup | Must be scoped for contacts, conversations/messages, notes/tags, and opportunities, added as HTTP Header Auth (`Authorization: Bearer <token>`). |
| Google Sheets OAuth2 Setup | Requires a connected Google account with permissions to read/write spreadsheet data and a tab named `Warranty Registry`. |
| OpenAI API Credentials | Required for the `gpt-4o-mini` node to dynamically draft contextual reminder messages and copy. |