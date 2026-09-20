Sync dental leads from Apify Google Maps to HighLevel CRM opportunities

https://n8nworkflows.xyz/workflows/sync-dental-leads-from-apify-google-maps-to-highlevel-crm-opportunities-19656


# Sync dental leads from Apify Google Maps to HighLevel CRM opportunities

### 1. Workflow Overview

This workflow automates the weekly extraction, filtering, and synchronization of dental clinic leads from Apify Google Maps scraping results into HighLevel (GoHighLevel) CRM. It ensures that local business data is cleanly integrated by checking existing records to prevent duplicates, upserting contact profiles, and automatically generating CRM opportunities for both new and existing leads. Additionally, an error-handling block captures any execution failures and reports them directly to a designated Slack channel.

The workflow logic is categorized into the following functional blocks:
- **1.1 Schedule & Data Ingestion:** Triggers weekly, requests raw Google Maps data from the Apify actor, and bounds the payload size.
- **1.2 Data Normalization & Validation:** Validates incoming records, filters out incomplete entries, and verifies whether a phone number is present.
- **1.3 CRM Contact Evaluation & Upsert:** Queries HighLevel for existing contacts, categorizes leads, and creates or updates records accordingly.
- **1.4 Pipeline & Opportunity Management (New Leads Branch):** Evaluates opportunity existence for newly upserted contacts and creates CRM opportunities if missing.
- **1.5 Pipeline & Opportunity Management (Existing Leads Branch):** Evaluates opportunity status for recognized existing contacts and creates CRM opportunities if missing.
- **1.6 Error Handling:** Listens for runtime errors across the workflow and posts descriptive alerts to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Data Ingestion
- **Overview:** Initializes the synchronization process on a regular weekly schedule, fetches raw scraped dental clinic listings from Apify, and restricts the processing batch size to prevent overloading downstream systems.
- **Nodes Involved:** `Weekly Schedule Trigger`, `Fetch Dental Clinics Data`, `Limit to 5 Leads`
- **Node Details:**
  - **Weekly Schedule Trigger**
    - *Type and technical role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration choices:* Set to execute on a recurring weekly interval.
    - *Key expressions or variables:* None.
    - *Input/Output connections:* Inputs: None | Outputs: `Fetch Dental Clinics Data`
    - *Edge cases:* Missed triggers due to system downtime (standard n8n catch-up behavior applies).
  - **Fetch Dental Clinics Data**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (Action)
    - *Configuration choices:* Sends an HTTP request to the Apify Google Maps actor using HTTP Header Authentication.
    - *Key expressions or variables:* Apify API token header configuration.
    - *Input/Output connections:* Inputs: `Weekly Schedule Trigger` | Outputs: `Limit to 5 Leads`
    - *Edge cases:* Apify API rate limits, authentication failures, or actor run timeouts.
  - **Limit to 5 Leads**
    - *Type and technical role:* `n8n-nodes-base.limit` (Data Transformation)
    - *Configuration choices:* Restricts the incoming data array to a maximum of 5 items per execution.
    - *Key expressions or variables:* Maximum items: `5`.
    - *Input/Output connections:* Inputs: `Fetch Dental Clinics Data` | Outputs: `If Phone Exists`
    - *Edge cases:* Empty datasets resulting in zero output items.

#### 1.2 Data Normalization & Validation
- **Overview:** Validates whether extracted listings contain minimum required contact details (such as a valid phone number) and normalizes the raw data fields into a consistent schema.
- **Nodes Involved:** `If Phone Exists`, `Flatten Google Maps Data`
- **Node Details:**
  - **If Phone Exists**
    - *Type and technical role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration choices:* Evaluates if the incoming item contains a valid phone property.
    - *Key expressions or variables:* Condition checking existence of lead phone data.
    - *Input/Output connections:* Inputs: `Limit to 5 Leads` | Outputs: `Flatten Google Maps Data` (True branch)
    - *Edge cases:* Malformed phone numbers or missing fields leading to skipped items.
  - **Flatten Google Maps Data**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Custom JavaScript execution to map nested JSON structures from Apify into flat, normalized properties (name, phone, website, domain, address, ratings) and drops entries lacking a name plus at least a phone or website.
    - *Key expressions or variables:* Standard JavaScript array mapping and filtering logic.
    - *Input/Output connections:* Inputs: `If Phone Exists` | Outputs: `Fetch Contacts from GHL`
    - *Edge cases:* Unexpected JSON schemas from the scraper causing JavaScript runtime reference errors.

#### 1.3 CRM Contact Evaluation & Upsert
- **Overview:** Interrogates HighLevel CRM to check for existing contact records based on phone numbers, categorizes leads into distinct operational paths, and upserts contact records.
- **Nodes Involved:** `Fetch Contacts from GHL`, `Categorize Leads`, `Check Lead Status`, `Post Lead Accounts to CRM`
- **Node Details:**
  - **Fetch Contacts from GHL**
    - *Type and technical role:* `n8n-nodes-base.highLevel` (Integration)
    - *Configuration choices:* Queries HighLevel API to search for contacts matching the normalized lead data. Configured with `alwaysOutputData: true`.
    - *Key expressions or variables:* Search parameters mapped from normalized lead properties.
    - *Input/Output connections:* Inputs: `Flatten Google Maps Data` | Outputs: `Categorize Leads`
    - *Edge cases:* API throttling or invalid location IDs.
  - **Categorize Leads**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Evaluates search results from HighLevel to label each incoming lead as either "new" or "existing".
    - *Key expressions or variables:* Conditional logic checking for matching contact IDs.
    - *Input/Output connections:* Inputs: `Fetch Contacts from GHL` | Outputs: `Check Lead Status`
    - *Edge cases:* Ambiguous search responses returning multiple contact matches.
  - **Check Lead Status**
    - *Type and technical role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration choices:* Splits workflow routing based on the lead categorization flag.
    - *Key expressions or variables:* Evaluates lead status string.
    - *Input/Output connections:* Inputs: `Categorize Leads` | Outputs: `Post Lead Accounts to CRM` (Branch 1 - New/Upsert), `Fetch Opportunities Phase 2` (Branch 2 - Existing)
    - *Edge cases:* Unhandled status values causing items to stall.
  - **Post Lead Accounts to CRM**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (Integration)
    - *Configuration choices:* Upserts contact information into HighLevel CRM via API request.
    - *Key expressions or variables:* Payload constructed from normalized lead fields.
    - *Input/Output connections:* Inputs: `Check Lead Status` | Outputs: `Fetch Opportunities from GHL`
    - *Edge cases:* Missing required CRM schema fields or permission errors.

#### 1.4 Pipeline & Opportunity Management (New Leads Branch)
- **Overview:** For newly processed leads, fetches existing opportunities from HighLevel, evaluates whether an opportunity is already open in the target pipeline, and creates one if missing.
- **Nodes Involved:** `Fetch Opportunities from GHL`, `Categorize Opportunities`, `Evaluate Opportunity Status`, `Post Opportunity to CRM`
- **Node Details:**
  - **Fetch Opportunities from GHL**
    - *Type and technical role:* `n8n-nodes-base.highLevel` (Integration)
    - *Configuration choices:* Retrieves opportunity listings tied to the contact/phone number. Configured with `alwaysOutputData: true`.
    - *Key expressions or variables:* Contact or phone identifier from previous node.
    - *Input/Output connections:* Inputs: `Post Lead Accounts to CRM` | Outputs: `Categorize Opportunities`
    - *Edge cases:* Empty response structures when no opportunities exist.
  - **Categorize Opportunities**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Inspects fetched opportunities to determine if an entry already exists within the configured pipeline and stage.
    - *Key expressions or variables:* JavaScript array searching against target pipeline IDs.
    - *Input/Output connections:* Inputs: `Fetch Opportunities from GHL` | Outputs: `Evaluate Opportunity Status`
    - *Edge cases:* Array iteration errors if API output format changes unexpectedly.
  - **Evaluate Opportunity Status**
    - *Type and technical role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration choices:* Checks boolean result from categorization indicating opportunity absence.
    - *Key expressions or variables:* Boolean check on opportunity existence.
    - *Input/Output connections:* Inputs: `Categorize Opportunities` | Outputs: `Post Opportunity to CRM`
    - *Edge cases:* False positives routing items incorrectly.
  - **Post Opportunity to CRM**
    - *Type and technical role:* `n8n-nodes-base.highLevel` (Integration)
    - *Configuration choices:* Creates a new opportunity record in the specified HighLevel pipeline and stage.
    - *Key expressions or variables:* Pipeline ID, Stage ID, Location ID, and Contact ID.
    - *Input/Output connections:* Inputs: `Evaluate Opportunity Status` | Outputs: None
    - *Edge cases:* Invalid stage IDs or pipeline configuration mismatches.

#### 1.5 Pipeline & Opportunity Management (Existing Leads Branch)
- **Overview:** Handles leads identified as existing contacts, checks for active opportunities in HighLevel, and creates a new opportunity if none is found.
- **Nodes Involved:** `Fetch Opportunities Phase 2`, `Categorize Opportunities Phase2`, `Verify Opportunity Status`, `Post Second Opportunity to CRM`
- **Node Details:**
  - **Fetch Opportunities Phase 2**
    - *Type and technical role:* `n8n-nodes-base.highLevel` (Integration)
    - *Configuration choices:* Queries HighLevel for opportunities associated with the existing contact. Configured with `alwaysOutputData: true`.
    - *Key expressions or variables:* Contact ID or phone parameter.
    - *Input/Output connections:* Inputs: `Check Lead Status` | Outputs: `Categorize Opportunities Phase2`
    - *Edge cases:* Timeouts or API rate-limiting during high-volume executions.
  - **Categorize Opportunities Phase2**
    - *Type and technical role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration choices:* Evaluates existing opportunities to identify whether the target pipeline contains an open entry.
    - *Key expressions or variables:* Custom script evaluating pipeline match.
    - *Input/Output connections:* Inputs: `Fetch Opportunities Phase 2` | Outputs: `Verify Opportunity Status`
    - *Edge cases:* Syntax or reference errors within custom JavaScript code.
  - **Verify Opportunity Status**
    - *Type and technical role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration choices:* Routes execution based on whether an opportunity needs to be created.
    - *Key expressions or variables:* Conditional evaluation flag.
    - *Input/Output connections:* Inputs: `Categorize Opportunities Phase2` | Outputs: `Post Second Opportunity to CRM`
    - *Edge cases:* Unmatched evaluation rules halting the branch.
  - **Post Second Opportunity to CRM**
    - *Type and technical role:* `n8n-nodes-base.highLevel` (Integration)
    - *Configuration choices:* Submits a request to HighLevel to create a new opportunity for the existing contact.
    - *Key expressions or variables:* Target pipeline ID, stage ID, and contact reference.
    - *Input/Output connections:* Inputs: `Verify Opportunity Status` | Outputs: None
    - *Edge cases:* Authentication token expiration or invalid CRM parameters.

#### 1.6 Error Handling
- **Overview:** Listens for unhandled workflow errors and dispatches detailed notifications to a configured Slack channel.
- **Nodes Involved:** `On Error Trigger`, `Post Error to Slack`
- **Node Details:**
  - **On Error Trigger**
    - *Type and technical role:* `n8n-nodes-base.errorTrigger` (Trigger)
    - *Configuration choices:* Automatically activates whenever any node in the workflow encounters a fatal error.
    - *Key expressions or variables:* Execution context variables containing error details and failed node names.
    - *Input/Output connections:* Inputs: None | Outputs: `Post Error to Slack`
    - *Edge cases:* Infinite loops if error notification nodes fail (mitigated by n8n internal safeguards).
  - **Post Error to Slack**
    - *Type and technical role:* `n8n-nodes-base.slack` (Integration)
    - *Configuration choices:* Formats and posts an alert message containing error diagnostics to a specific Slack channel using Slack OAuth2 or Bot credentials.
    - *Key expressions or variables:* `$json.execution.error.message`, `$json.execution.error.node.name`.
    - *Input/Output connections:* Inputs: `On Error Trigger` | Outputs: None
    - *Edge cases:* Slack API outages or invalid webhook/channel configurations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Schedule Trigger | scheduleTrigger | Trigger workflow weekly | None | Fetch Dental Clinics Data | |
| Fetch Dental Clinics Data | httpRequest | Fetch data from Apify actor | Weekly Schedule Trigger | Limit to 5 Leads | |
| Limit to 5 Leads | limit | Restrict batch size to 5 items | Fetch Dental Clinics Data | If Phone Exists | |
| If Phone Exists | if | Validate presence of phone data | Limit to 5 Leads | Flatten Google Maps Data | |
| Flatten Google Maps Data | code | Normalize and clean scraped data | If Phone Exists | Fetch Contacts from GHL | |
| Fetch Contacts from GHL | highLevel | Search existing CRM contacts | Flatten Google Maps Data | Categorize Leads | |
| Categorize Leads | code | Label leads as new or existing | Fetch Contacts from GHL | Check Lead Status | |
| Check Lead Status | if | Route workflow by lead status | Categorize Leads | Post Lead Accounts to CRM, Fetch Opportunities Phase 2 | |
| Post Lead Accounts to CRM | httpRequest | Upsert contact data into CRM | Check Lead Status | Fetch Opportunities from GHL | |
| Fetch Opportunities from GHL | highLevel | Retrieve CRM opportunities for new leads | Post Lead Accounts to CRM | Categorize Opportunities | |
| Categorize Opportunities | code | Check opportunity existence | Fetch Opportunities from GHL | Evaluate Opportunity Status | |
| Evaluate Opportunity Status | if | Evaluate if opportunity creation is needed | Categorize Opportunities | Post Opportunity to CRM | |
| Post Opportunity to CRM | highLevel | Create opportunity for new lead | Evaluate Opportunity Status | None | |
| Fetch Opportunities Phase 2 | highLevel | Retrieve CRM opportunities for existing leads | Check Lead Status | Categorize Opportunities Phase2 | |
| Categorize Opportunities Phase2 | code | Check opportunity existence for existing leads | Fetch Opportunities Phase 2 | Verify Opportunity Status | |
| Verify Opportunity Status | if | Evaluate if opportunity creation is needed | Categorize Opportunities Phase2 | Post Second Opportunity to CRM | |
| Post Second Opportunity to CRM | highLevel | Create opportunity for existing lead | Verify Opportunity Status | None | |
| On Error Trigger | errorTrigger | Catch execution errors | None | Post Error to Slack | |
| Post Error to Slack | slack | Send failure alert to Slack channel | On Error Trigger | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the workflow manually in n8n:

1. **Create Triggers and Error Handling Nodes:**
   - Add a **Schedule Trigger** node (`Weekly Schedule Trigger`). Set execution interval to weekly.
   - Add an **Error Trigger** node (`On Error Trigger`).
2. **Set Up Data Ingestion:**
   - Add an **HTTP Request** node (`Fetch Dental Clinics Data`). Configure authentication via HTTP Header Auth (Apify API token) and set the endpoint URL to your Apify Google Maps actor run endpoint. Connect `Weekly Schedule Trigger` to this node.
   - Add a **Limit** node (`Limit to 5 Leads`). Set the maximum items parameter to `5`. Connect `Fetch Dental Clinics Data` to this node.
3. **Configure Validation and Normalization:**
   - Add an **If** node (`If Phone Exists`). Configure it to verify that the incoming item contains a valid phone property. Connect `Limit to 5 Leads` to this node.
   - Add a **Code** node (`Flatten Google Maps Data`). Insert JavaScript code to parse, map, and filter the raw Apify JSON fields into a flat schema (retaining name, phone, website, domain, address, rating; discarding entries lacking a name and either a phone or website). Connect the True output of `If Phone Exists` to this node.
4. **Implement CRM Contact Search and Categorization:**
   - Add a **HighLevel** node (`Fetch Contacts from GHL`). Configure HighLevel OAuth2 credentials, set resource to `Contact`, operation to `Get Many` or search, and map the search parameter to the normalized phone number. Enable **Always Output Data**. Connect `Flatten Google Maps Data` to this node.
   - Add a **Code** node (`Categorize Leads`). Write JavaScript logic to compare fetched contact records against incoming lead data, adding a classification status property (`new` or `existing`). Connect `Fetch Contacts from GHL` to this node.
   - Add an **If** node (`Check Lead Status`). Configure expressions to evaluate the lead status property. Connect `Categorize Leads` to this node.
5. **Build the New Leads (Upsert & Opportunity) Branch:**
   - Add an **HTTP Request** node (`Post Lead Accounts to CRM`) or HighLevel Upsert node. Configure the request payload using normalized lead properties to create or update the contact in HighLevel. Connect the primary output of `Check Lead Status` to this node.
   - Add a **HighLevel** node (`Fetch Opportunities from GHL`). Set operation to retrieve opportunities associated with the contact. Enable **Always Output Data**. Connect `Post Lead Accounts to CRM` to this node.
   - Add a **Code** node (`Categorize Opportunities`). Implement JavaScript to check if an opportunity exists within the target pipeline and stage. Connect `Fetch Opportunities from GHL` to this node.
   - Add an **If** node (`Evaluate Opportunity Status`). Configure condition to check if opportunity creation is required. Connect `Categorize Opportunities` to this node.
   - Add a **HighLevel** node (`Post Opportunity to CRM`). Set resource to `Opportunity`, operation to `Create`, and configure parameters with your target `locationId`, `pipelineId`, and `stageId`. Connect the True output of `Evaluate Opportunity Status` to this node.
6. **Build the Existing Leads Opportunity Branch:**
   - Add a **HighLevel** node (`Fetch Opportunities Phase 2`). Set operation to retrieve opportunities for the existing contact. Enable **Always Output Data**. Connect the secondary output of `Check Lead Status` to this node.
   - Add a **Code** node (`Categorize Opportunities Phase2`). Implement verification logic to check pipeline presence. Connect `Fetch Opportunities Phase 2` to this node.
   - Add an **If** node (`Verify Opportunity Status`). Configure condition to check if an opportunity should be created. Connect `Categorize Opportunities Phase2` to this node.
   - Add a **HighLevel** node (`Post Second Opportunity to CRM`). Set resource to `Opportunity`, operation to `Create`, and supply your `locationId`, `pipelineId`, and `stageId`. Connect the True output of `Verify Opportunity Status` to this node.
7. **Finalize Error Reporting:**
   - Add a **Slack** node (`Post Error to Slack`). Configure Slack OAuth2 or Bot credentials, select the destination channel, and set the message content to display execution error details using expressions: `={{ $json.execution.error.message }}` and `={{ $json.execution.error.node.name }}`. Connect `On Error Trigger` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video walkthrough and reference guide | https://youtu.be/FUI6QNU9IJE |