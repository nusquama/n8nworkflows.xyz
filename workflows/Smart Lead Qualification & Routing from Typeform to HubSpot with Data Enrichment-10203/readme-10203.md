Smart Lead Qualification & Routing from Typeform to HubSpot with Data Enrichment

https://n8nworkflows.xyz/workflows/smart-lead-qualification---routing-from-typeform-to-hubspot-with-data-enrichment-10203


# Smart Lead Qualification & Routing from Typeform to HubSpot with Data Enrichment

### 1. Workflow Overview

This workflow automates the lead qualification and routing process, starting from form submissions via Typeform, enriching the data with external APIs, scoring the leads, and finally routing them into HubSpot CRM with appropriate lead stages. It also includes nurture stage follow-up with delay, revalidation, Slack alerts, and logging for lower-priority leads.

Logical blocks:

- **1.1 Input Reception**: Receiving lead submissions through a webhook from Typeform.
- **1.2 Lead Data Extraction**: Parsing and extracting relevant lead info (name, email, company).
- **1.3 Email Verification**: Verifying lead email validity with Hunter.io API.
- **1.4 Domain Extraction & Company Enrichment**: Extracting email domain and enriching company data via Abstract API.
- **1.5 Lead Scoring & Routing Decision**: Scoring leads based on enriched data and routing them according to priority.
- **1.6 HubSpot Contact Creation**: Creating/Updating contacts in HubSpot with enriched data and lead stage.
- **1.7 Nurture Stage Follow-up**: For lower-priority leads, implementing a delay, revalidation with HubSpot, Slack notification, and Google Sheets logging.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Entry point to capture incoming leads from Typeform via a webhook.
- **Nodes Involved:** Webhook Trigger
- **Node Details:**
  - **Webhook Trigger**
    - Type: Webhook node, HTTP POST method.
    - Configuration: Listens for POST requests at a specified webhook path (`/webhook/new-lead-webhook`).
    - Inputs: External HTTP request (Typeform form submission).
    - Outputs: JSON payload containing Typeform form response.
    - Edge cases: Missing or malformed data, webhook not triggered if path is incorrect or unauthorized requests.
    - Sticky Note: Describes webhook purpose and path.

#### 1.2 Lead Data Extraction

- **Overview:** Parses the incoming Typeform JSON payload to extract key lead fields and derive the email domain.
- **Nodes Involved:** Extract Lead Data
- **Node Details:**
  - **Extract Lead Data**
    - Type: Function node.
    - Configuration: Custom JavaScript parsing `body.form_response` to extract first name, last name, email, company, and email domain.
    - Expressions: Uses multiple checks on field titles and answer types.
    - Inputs: Output from Webhook Trigger.
    - Outputs: JSON object with extracted fields and domain.
    - Edge cases: Missing fields, unexpected JSON schema changes, empty answers array.
    - Sticky Note: Explains the data parsing logic and domain derivation.

#### 1.3 Email Verification

- **Overview:** Uses Hunter.io API to verify the validity and deliverability of the extracted email.
- **Nodes Involved:** Hunter.io Verification
- **Node Details:**
  - **Hunter.io Verification**
    - Type: HTTP Request node.
    - Configuration: GET request to Hunter.io email verifier endpoint with email and API key parameters.
    - Inputs: Extracted email from previous node.
    - Outputs: Verification status and details.
    - Edge cases: API key misconfiguration, rate limits, invalid email input, network errors.
    - Sticky Note: Confirms email verification purpose.

#### 1.4 Domain Extraction & Company Enrichment

- **Overview:** Extracts domain from email (if not extracted previously) and enriches company data via Abstract API.
- **Nodes Involved:** Extract Domain for Abstract, Abstract Company Enrichment
- **Node Details:**
  - **Extract Domain for Abstract**
    - Type: Code node.
    - Configuration: Extracts domain string from email field.
    - Inputs: Output from Hunter.io Verification (email field).
    - Outputs: JSON object containing domain.
    - Edge cases: Missing or malformed email.
    - Sticky Note: Describes domain extraction.
  - **Abstract Company Enrichment**
    - Type: HTTP Request node.
    - Configuration: GET request to Abstract API with domain and API key query parameters.
    - Inputs: Extracted domain.
    - Outputs: Enriched company data including industry, employee count, location, website.
    - Edge cases: API key errors, invalid domain, rate limits, network issues.
    - Sticky Note: Details enrichment purpose.

#### 1.5 Lead Scoring & Routing Decision

- **Overview:** Scores the lead based on enriched data and routes leads into Qualified or Nurture paths depending on score threshold.
- **Nodes Involved:** Lead Scoring & Routing Logic, IF High Priority
- **Node Details:**
  - **Lead Scoring & Routing Logic**
    - Type: Function node.
    - Configuration: Custom JS scoring logic:
      - Company size (employees count weight)
      - Target countries (US, UK)
      - Industry match (software, IT)
      - Email domain type (rejects free email domains)
      - LinkedIn presence
    - Calculates total lead score and assigns tier: Cold, Warm, Hot.
    - Inputs: Enriched data from Abstract Company Enrichment.
    - Outputs: Lead data enriched with score, tier, timestamp.
    - Edge cases: Missing company info, missing fields in enrichment, unexpected data types.
    - Sticky Note: Explains scoring criteria.
  - **IF High Priority**
    - Type: If node (conditional).
    - Configuration: Checks if lead score >= 70.
    - Inputs: Lead scoring output.
    - Outputs: Two branches - TRUE for Qualified, FALSE for Nurture.
    - Edge cases: Missing or invalid score, expression evaluation errors.
    - Sticky Note: Describes routing logic.

#### 1.6 HubSpot Contact Creation

- **Overview:** Creates or updates contacts in HubSpot CRM with lead data and proper lead stage based on priority.
- **Nodes Involved:** HubSpot Create Contact (Qualified), HubSpot Create Contact (Nurture)
- **Node Details:**
  - **HubSpot Create Contact (Qualified)**
    - Type: HubSpot node.
    - Configuration: Creates contact with enriched data, sets lead stage to "Qualified".
    - Inputs: TRUE branch from IF High Priority.
    - Outputs: None (end branch).
    - Expressions: Maps fields from extracted and enriched data to HubSpot contact properties.
    - Edge cases: API auth errors, duplicate contacts, invalid data fields.
    - Sticky Note: Describes Qualified contact creation.
  - **HubSpot Create Contact (Nurture)**
    - Type: HubSpot node.
    - Configuration: Creates contact with enriched data, sets lead stage to "Nurture".
    - Inputs: FALSE branch from IF High Priority.
    - Outputs: Leads to nurture follow-up cycle.
    - Expressions: Similar field mappings as qualified branch.
    - Edge cases: Same as above.
    - Sticky Note: Describes Nurture contact creation.

#### 1.7 Nurture Stage Follow-up

- **Overview:** For leads in nurture, implements a delay, revalidates with HubSpot, sends Slack alert, and logs the lead in Google Sheets.
- **Nodes Involved:** Wait (3 Days) → ⏳ Delay for Nurture Recheck, HubSpot API Check → 🔍 HubSpot Revalidation, Slack Notification → 📨 Nurture Alert, Google Sheets Logging → 📊 Nurture Log
- **Node Details:**
  - **Wait (3 Days) → ⏳ Delay for Nurture Recheck**
    - Type: Wait node.
    - Configuration: Delays workflow for 10 seconds (test) or 3 days (production).
    - Inputs: From HubSpot Create Contact (Nurture).
    - Outputs: Continues to HubSpot Revalidation.
    - Edge cases: Workflow timeouts, node misconfiguration.
    - Sticky Note: Explains purpose of delay and test vs production settings.
  - **HubSpot API Check → 🔍 HubSpot Revalidation**
    - Type: HTTP Request node.
    - Configuration: POST request to HubSpot API to query contacts with lead_stage = Nurture.
    - Inputs: After delay.
    - Outputs: Lead data for nurture stage leads.
    - Edge cases: API key errors, rate limits, no nurture leads found.
    - Sticky Note: Explains revalidation purpose.
  - **Slack Notification → 📨 Nurture Alert**
    - Type: Slack node.
    - Configuration: Sends formatted alert with company, email, score, and timestamp.
    - Inputs: Output from HubSpot Revalidation.
    - Outputs: None.
    - Edge cases: Slack auth errors, message formatting issues.
    - Sticky Note: Describes Slack alert content.
  - **Google Sheets Logging → 📊 Nurture Log**
    - Type: Google Sheets node.
    - Configuration: Appends lead info (date, company, email, score, stage) to a specified sheet.
    - Inputs: Output from HubSpot Revalidation.
    - Outputs: None.
    - Edge cases: Google Sheets API errors, missing spreadsheet ID, permission issues.
    - Sticky Note: Describes logging purpose.

---

### 3. Summary Table

| Node Name                            | Node Type           | Functional Role                        | Input Node(s)                   | Output Node(s)                                   | Sticky Note                                                                                                     |
|------------------------------------|---------------------|-------------------------------------|--------------------------------|-------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook Trigger                    | Webhook             | Entry point for Typeform submissions | None                           | Extract Lead Data                               | 🟢 1. Webhook Trigger Entry Point — Receives form submissions from Typeform. Webhook path: `/webhook/new-lead-webhook` |
| Extract Lead Data                  | Function            | Parses lead info from Typeform JSON | Webhook Trigger                | Hunter.io Verification                          | 🟦 2. Extract Lead Data Parses Typeform JSON to extract key fields and derive domain                            |
| Hunter.io Verification            | HTTP Request        | Verifies email validity with Hunter.io | Extract Lead Data              | Extract Domain for Abstract                      | 🟩 3. Hunter.io Verification Verifies lead email validity                                                     |
| Extract Domain for Abstract       | Code                | Extracts email domain for enrichment | Hunter.io Verification         | Abstract Company Enrichment                      | 🟨 4. Extract Domain for Abstract Extracts domain from email                                                  |
| Abstract Company Enrichment       | HTTP Request        | Enriches company data via Abstract API | Extract Domain for Abstract    | Lead Scoring & Routing Logic                     | 🟧 5. Abstract Company Enrichment Fetches company details                                                     |
| Lead Scoring & Routing Logic      | Function            | Calculates lead score and assigns tier | Abstract Company Enrichment    | IF High Priority                                | 🟪 6. Lead Scoring & Routing Logic Scores lead and assigns tier                                               |
| IF High Priority                 | If                  | Routes leads based on score threshold | Lead Scoring & Routing Logic   | HubSpot Create Contact (Qualified), HubSpot Create Contact (Nurture) | 🔵 7. IF High Priority Decision node routing Qualified vs Nurture leads                                      |
| HubSpot Create Contact (Qualified) | HubSpot             | Creates Qualified contacts in HubSpot | IF High Priority (true branch) | None                                            | 🟣 7.1 HubSpot Create Contact (Qualified) Creates “Qualified” contacts with enriched data                      |
| HubSpot Create Contact (Nurture)  | HubSpot             | Creates Nurture contacts in HubSpot  | IF High Priority (false branch) | Wait (3 Days) → ⏳ Delay for Nurture Recheck    | 🟤 7.2 HubSpot Create Contact (Nurture) Creates “Nurture” stage contacts                                       |
| Wait (3 Days) → ⏳ Delay for Nurture Recheck | Wait                | Delays workflow for nurture revalidation | HubSpot Create Contact (Nurture) | HubSpot API Check → 🔍 HubSpot Revalidation     | 🟠 7.2.1 Wait node for delay before nurture revalidation                                                      |
| HubSpot API Check → 🔍 HubSpot Revalidation | HTTP Request        | Queries nurture leads in HubSpot CRM | Wait (3 Days)                 | Google Sheets Logging → 📊 Nurture Log, Slack Notification → 📨 Nurture Alert | 🔴 7.2.2 HubSpot revalidation queries nurture contacts                                                       |
| Slack Notification → 📨 Nurture Alert | Slack               | Sends Slack alerts for nurture leads | HubSpot API Check              | None                                            | 🟢 7.2.2.1 Slack alert for nurture leads                                                                     |
| Google Sheets Logging → 📊 Nurture Log | Google Sheets        | Logs nurture lead info in Google Sheets | HubSpot API Check              | None                                            | 🟣 7.2.2.2 Logs nurture leads in Google Sheets                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger node**
   - Type: Webhook
   - HTTP Method: POST
   - Configure path (e.g. `/webhook/new-lead-webhook`)
   - No authentication required (handle externally if needed)

2. **Add Function node "Extract Lead Data"**
   - Input: From Webhook Trigger
   - Use custom JavaScript code to parse `body.form_response`
   - Extract first name, last name, email, company from answers and fields
   - Derive email domain from extracted email
   - Output JSON with fields: `first_name`, `last_name`, `email`, `company`, `domain`

3. **Add HTTP Request node "Hunter.io Verification"**
   - Input: From Extract Lead Data
   - Method: GET
   - URL: `https://api.hunter.io/v2/email-verifier`
   - Query parameters:
     - `email` = `{{$json.email}}`
     - `api_key` = Your Hunter.io API Key
   - Handle errors for invalid or missing email

4. **Add Code node "Extract Domain for Abstract"**
   - Input: From Hunter.io Verification
   - JavaScript to extract domain from email field in response
   - Output JSON with `domain`

5. **Add HTTP Request node "Abstract Company Enrichment"**
   - Input: From Extract Domain for Abstract
   - Method: GET
   - URL: `https://companyenrichment.abstractapi.com/v1/`
   - Query parameters:
     - `api_key` = Your Abstract API key
     - `domain` = `{{$json.domain}}`
   - Expected JSON response with company info (industry, employees_count, country, name, linkedin_url, etc.)

6. **Add Function node "Lead Scoring & Routing Logic"**
   - Input: From Abstract Company Enrichment
   - JavaScript scoring logic:
     - Score based on employees_count thresholds
     - Add points for target countries (US, UK)
     - Add points for target industries (Software, Internet, IT)
     - Add points if email domain is not free email (gmail, yahoo)
     - Add points if LinkedIn URL present
   - Assign lead_tier: Hot (>=70), Warm (40-69), Cold (<40)
   - Append score, tier, timestamp to JSON output

7. **Add If node "IF High Priority"**
   - Input: From Lead Scoring & Routing Logic
   - Condition: `lead_score >= 70`
   - TRUE branch: Qualified path
   - FALSE branch: Nurture path

8. **Add HubSpot node "HubSpot Create Contact (Qualified)"**
   - Input: TRUE branch from IF High Priority
   - Operation: Create or update contact
   - Map fields: email, firstName, lastName, companyName, websiteUrl, linkedinUrl, city, country, industry, lead_score, lead_tier, lead_stage = "Qualified"
   - Configure HubSpot OAuth2 credentials

9. **Add HubSpot node "HubSpot Create Contact (Nurture)"**
   - Input: FALSE branch from IF High Priority
   - Same field mappings as Qualified node, but lead_stage = "Nurture"

10. **Add Wait node "Wait (3 Days) → ⏳ Delay for Nurture Recheck"**
    - Input: From HubSpot Create Contact (Nurture)
    - Set delay to 10 seconds for testing or 3 days for production

11. **Add HTTP Request node "HubSpot API Check → 🔍 HubSpot Revalidation"**
    - Input: From Wait node
    - Method: POST
    - URL: HubSpot CRM search endpoint (e.g., `https://api.hubapi.com/crm/v3/objects/contacts/search`)
    - JSON body:
      ```json
      {
        "filterGroups": [{
          "filters": [{
            "propertyName": "lead_stage",
            "operator": "EQ",
            "value": "Nurture"
          }]
        }],
        "properties": ["email", "lead_score", "lead_stage", "firstname", "company"]
      }
      ```
    - Use HubSpot OAuth2 credentials

12. **Add Slack node "Slack Notification → 📨 Nurture Alert"**
    - Input: From HubSpot API Check
    - Slack message text with template including company, email, lead score, timestamp
    - Configure Slack credentials and channel

13. **Add Google Sheets node "Google Sheets Logging → 📊 Nurture Log"**
    - Input: From HubSpot API Check
    - Operation: Append row to sheet
    - Map fields: Date (ISO string), Company, Email, Lead Score, Stage
    - Configure Google Sheets credentials, document ID, and sheet name

14. **Connect nodes in sequence as per above logic**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| This workflow automates a smart lead routing & qualification system integrating Typeform, Hunter.io, Abstract API, HubSpot CRM, Slack, and Google Sheets.    | Workflow Summary Sticky Note                                                |
| Delay in the nurture follow-up is set to 10 seconds for testing purposes; change to 3 days for production deployment.                                        | Wait node Sticky Note                                                       |
| HubSpot contact creation nodes are configured for version 2.2 of the HubSpot node, ensure credentials are OAuth2 and have sufficient API scopes.              | HubSpot nodes                                                              |
| Slack notification includes lead details to alert the sales/support team immediately on nurture leads pending follow-up.                                   | Slack node Sticky Note                                                      |
| Google Sheets logging appends nurture follow-up leads to a dedicated spreadsheet for tracking and reporting.                                                | Google Sheets node Sticky Note                                              |
| The workflow depends on external API keys: Hunter.io and Abstract API. Keep API keys secure and monitor usage limits.                                        | API Request nodes                                                           |
| Typeform webhook must be configured to send POST requests to the n8n webhook URL to trigger the workflow.                                                    | Webhook Trigger node                                                        |
| Lead scoring logic is customizable; adjust scoring weights or target industries/countries as per business requirements.                                     | Lead Scoring & Routing Logic node                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. All data processed is legal and public. The workflow respects content policies and contains no illegal, offensive, or protected elements.