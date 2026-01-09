Enrich contacts with Wiza and sync results to Airtable and HubSpot

https://n8nworkflows.xyz/workflows/enrich-contacts-with-wiza-and-sync-results-to-airtable-and-hubspot-12343


# Enrich contacts with Wiza and sync results to Airtable and HubSpot

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (provided):** Enrich contacts with Wiza and sync results to Airtable and HubSpot  
**Workflow name (JSON):** Full Contact Enrichment Workflow

**Purpose:**  
This workflow collects lead data via an n8n Form, enriches it using Wiza (email/phone/LinkedIn depending on the user’s choice), normalizes the results into a single “lead” object, and syncs the enriched lead into:
- **HubSpot** (create/update contact)
- **Google Sheets** (append or update row)
- **n8n Data Table** (insert row)

It also returns a **success** or **failure** completion page back to the user.

### 1.1 Input Reception & User Intent Routing
- User submits a form indicating what enrichment they want (email/phone/linkedin/all).
- A Switch node routes to the correct Wiza operation(s).

### 1.2 Enrichment via Wiza + Aggregation
- One or more Wiza nodes run (Email Finder / Phone Finder / Linkedin Finder).
- A Merge node combines up to three enrichment results.

### 1.3 Status Gate (Failed vs Finished)
- A second Switch checks Wiza status and routes either to failure completion or to data normalization.

### 1.4 Normalize + Sync to Systems + Success UI
- A Code node produces a single normalized schema (first/last name, email, phone, etc.).
- Data is sent in parallel to HubSpot, Google Sheets, and n8n Data Table.
- A success completion form is shown after Sheets step completes.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Routing
**Overview:** Collects lead details and the desired enrichment type, then routes execution to one or more Wiza enrichment operations.

**Nodes involved:**
- On form submission
- Switch

#### Node: **On form submission**
- **Type / role:** `Form Trigger` (`n8n-nodes-base.formTrigger`) — entry point; receives user input.
- **Configuration (interpreted):**
  - Form title: **“Linkedin”**
  - Fields:
    - Dropdown **“What Are You Searching For?”** (required): Find Email / Find Phone / Find Linkedin / Find All
    - Optional text fields: Full Name, Company Name, Email, Phone, Linkedin
- **Key variables used:** Outputs fields as `$json[...]`, notably:
  - `$json["What Are You Searching For?"]`
- **Connections:**
  - Output → **Switch**
- **Version-specific:** typeVersion **2.3**
- **Edge cases / failures:**
  - Missing required dropdown stops submission.
  - If users leave identifying fields empty, Wiza may fail to enrich or return low-quality results.

#### Node: **Switch**
- **Type / role:** `Switch` (`n8n-nodes-base.switch`) — routes based on user selection.
- **Configuration (interpreted):**
  - 4 rules comparing:
    - `={{ $json['What Are You Searching For?'] }}` equals `Find Email` / `Find Phone` / `Find Linkedin` / `Find All`
  - Each rule uses “rename output” so outputs are labeled.
- **Connections:**
  - “Find Email” → **Email Finder**
  - “Find Phone” → **Phone Finder**
  - “Find Linkedin” → **Linkedin Finder**
  - “Find All” → **Email Finder**, **Phone Finder**, **Linkedin Finder** (fan-out)
- **Version-specific:** typeVersion **3.2**
- **Edge cases / failures:**
  - If the dropdown value differs (extra spaces / localization), no rule matches → downstream nodes won’t run.

---

### Block 2 — Wiza Enrichment & Merge
**Overview:** Calls Wiza for the chosen enrichment type(s), then merges the multiple results into one combined stream for downstream processing.

**Nodes involved:**
- Email Finder
- Phone Finder
- Linkedin Finder
- Merge

#### Node: **Email Finder**
- **Type / role:** Wiza node (`@wizaco/n8n-nodes-wiza.wiza`) — performs email enrichment.
- **Configuration (interpreted):**
  - Operation: default (implied **email finder**)
  - Additional fields: none set
  - Credentials: **Wiza account**
- **Connections:**
  - Input ← **Switch** (Find Email, Find All)
  - Output → **Merge** (input index 0)
- **Version-specific:** typeVersion **1**
- **Edge cases / failures:**
  - Auth/credit limits on Wiza
  - Insufficient input data (e.g., only company name without person identifiers)
  - Rate limiting / transient Wiza API errors
  - Response schema differences: could return `email`, or an `emails[]` array (handled later in Clean Up)

#### Node: **Phone Finder**
- **Type / role:** Wiza node — performs phone enrichment.
- **Configuration (interpreted):**
  - Operation: **phoneFinder**
  - Credentials: **Wiza account**
- **Connections:**
  - Input ← **Switch** (Find Phone, Find All)
  - Output → **Merge** (input index 1)
- **Version-specific:** typeVersion **1**
- **Edge cases / failures:**
  - Same categories as above; may return phones under `phone_number` or `phones[]`.

#### Node: **Linkedin Finder**
- **Type / role:** Wiza node — finds LinkedIn profile.
- **Configuration (interpreted):**
  - Operation: **linkedinFinder**
  - Credentials: **Wiza account**
- **Connections:**
  - Input ← **Switch** (Find Linkedin, Find All)
  - Output → **Merge** (input index 2)
- **Version-specific:** typeVersion **1**
- **Edge cases / failures:**
  - LinkedIn URL matching issues, ambiguous identities, or incomplete Wiza results.

#### Node: **Merge**
- **Type / role:** `Merge` (`n8n-nodes-base.merge`) — collects up to 3 inputs into one combined item list.
- **Configuration (interpreted):**
  - Number of inputs: **3**
  - (Mode not specified in JSON; in practice, this configuration typically merges streams so all results can be processed together downstream.)
- **Connections:**
  - Inputs ← Email Finder (0), Phone Finder (1), Linkedin Finder (2)
  - Output → **Switch1**
- **Version-specific:** typeVersion **3.2**
- **Edge cases / failures:**
  - When only one branch runs (e.g., Find Email), Merge behavior depends on its mode; misconfiguration can cause waiting for missing branches.
  - If one Wiza call errors and the others succeed, Merge may not emit items (depends on error handling settings, not shown here).

---

### Block 3 — Status Gate (Failed vs Finished)
**Overview:** Routes results based on the Wiza status field to either a failure completion message or the normalization/sync block.

**Nodes involved:**
- Switch1
- Failed To Find Enrichment

#### Node: **Switch1**
- **Type / role:** `Switch` — checks Wiza job/result status.
- **Configuration (interpreted):**
  - If `={{ $json.status }}` equals `failed` → output “Failed”
  - If `={{ $json.status }}` equals `finished` → output “Finished”
- **Connections:**
  - “Failed” → **Failed To Find Enrichment**
  - “Finished” → **Clean Up**
- **Version-specific:** typeVersion **3.2**
- **Edge cases / failures:**
  - If Wiza returns status values not exactly `failed`/`finished` (e.g., `success`, `completed`, uppercase), no rule matches and execution stops.
  - If `status` is missing (schema change), expression evaluates to `undefined`.

#### Node: **Failed To Find Enrichment**
- **Type / role:** `Form` (`n8n-nodes-base.form`) — completion page for failure.
- **Configuration (interpreted):**
  - Operation: **completion**
  - Title: **Failed To Find Enrichment Data**
  - Message: **The Wiza Back-End Could Not Find any Enrichment Data from the provided information**
- **Connections:** none (terminal)
- **Version-specific:** typeVersion **2.3**
- **Edge cases / failures:**
  - Uses a webhookId that is also used by the success form in this workflow; ensure n8n supports this pattern in your environment and that the UI flow behaves as expected.

---

### Block 4 — Normalize + Sync + Success Completion
**Overview:** Consolidates enrichment outputs into a normalized lead schema, then syncs to HubSpot, Google Sheets, and n8n Data Table. Finally shows a success completion message (after Sheets step).

**Nodes involved:**
- Clean Up
- Create or update a contact
- Append or update row in sheet
- Insert row
- Success Message Form

#### Node: **Clean Up**
- **Type / role:** `Code` (`n8n-nodes-base.code`) — transforms multiple Wiza results into one normalized object.
- **Configuration (interpreted):**
  - Takes all incoming items, merges them, and creates a single output item with fields:
    - `first_name`, `last_name` derived from `base.name`
    - `email` chosen from `emailRecord.email` or `emailRecord.emails[0].email`
    - `phone_number` chosen from `phoneRecord.phone_number` or `phoneRecord.phones[0].number`
    - `company_name` from `base.company`
    - `website` from `base.company_domain`
    - `location` from `base.location`
    - `job_title` from `base.title`
    - `linkedin_profile` from `base.linkedin_profile_url`
    - `company_url` from `base.company_domain`
  - It selects the “base” object as the first item (`all[0]`) and assumes name/company shared across results.
- **Key expressions/variables:** Uses JS optional chaining and array mapping:
  - `const all = items.map(i => i.json);`
  - `const base = all[0];`
- **Connections:**
  - Input ← **Switch1** (“Finished”)
  - Output → **Create or update a contact**, **Append or update row in sheet**, **Insert row** (parallel fan-out)
- **Version-specific:** typeVersion **2**
- **Edge cases / failures:**
  - If `items` is empty (e.g., Merge produced nothing), `all[0]` is `undefined` → runtime error when accessing `base.name`.
  - Names with single token: last_name becomes empty string (`""`) not null (because `join(" ")` returns `""`); may not be desired for HubSpot validation.
  - `company_url` duplicates `company_domain` (might need full URL with protocol).
  - If Wiza returns different field names, normalization may yield nulls.

#### Node: **Create or update a contact**
- **Type / role:** `HubSpot` (`n8n-nodes-base.hubspot`) — upserts a contact.
- **Configuration (interpreted):**
  - Authentication: **App Token**
  - Uses `email` as the primary identifier: `={{ $json.email }}`
  - Maps additional fields:
    - firstName, lastName, phoneNumber, jobTitle, city, companyName, linkedinUrl
- **Connections:**
  - Input ← **Clean Up**
  - Output: none connected (runs in parallel but not used downstream)
- **Version-specific:** typeVersion **2.2**
- **Edge cases / failures:**
  - If `email` is null/empty, HubSpot upsert will fail (email is required for contact identity in this node configuration).
  - App token scope limitations can cause 401/403.
  - Field naming: ensure these properties map to existing HubSpot contact properties (some portals customize fields).

#### Node: **Append or update row in sheet**
- **Type / role:** `Google Sheets` — appends a new row or updates an existing row.
- **Configuration (interpreted):**
  - Operation: **appendOrUpdate**
  - Spreadsheet: “Leads List” (documentId set)
  - Sheet/tab: “Leads 2025-09-30”
  - Matching column(s): **email**
  - Mapping mode: **auto map input data** against schema columns:
    - first_name, last_name, email, phone_number, company_name, website, location, job_title, linkedin_profile, company_url
- **Connections:**
  - Input ← **Clean Up**
  - Output → **Success Message Form**
- **Version-specific:** typeVersion **4.7**
- **Edge cases / failures:**
  - If `email` is null, appendOrUpdate with matching on email may append duplicates or error (behavior can vary).
  - OAuth token expiration/refresh issues.
  - Header/schema mismatch: if the sheet columns differ, auto-mapping may not map as intended.

#### Node: **Insert row**
- **Type / role:** `Data Table` (`n8n-nodes-base.dataTable`) — stores a record in an n8n internal Data Table.
- **Configuration (interpreted):**
  - Operation: Insert row into Data Table **“Leads”**
  - Mapping mode: “define below” (but no explicit field mappings are shown in the JSON excerpt)
  - Schema includes fields like name/title/location/linkedin/email/phone/company_* etc.
- **Connections:**
  - Input ← **Clean Up**
  - Output: none connected
- **Version-specific:** typeVersion **1**
- **Edge cases / failures:**
  - Without explicit mappings, inserted row may be empty or partially populated depending on node defaults.
  - Data Table feature availability depends on n8n edition/version and project configuration.

#### Node: **Success Message Form**
- **Type / role:** `Form` completion — final user feedback on success.
- **Configuration (interpreted):**
  - Title: **Success!**
  - Message: **Your new lead has been successfully added to your CRM**
- **Connections:**
  - Input ← **Append or update row in sheet**
  - Output: none (terminal)
- **Version-specific:** typeVersion **2.3**
- **Edge cases / failures:**
  - If HubSpot or Data Table insert fails but Sheets succeeds, user still sees success (because success is only chained from Sheets).
  - Uses same webhookId value as the failure completion form; confirm this behaves correctly in your deployment.

---

### Block 5 — Documentation / Notes (Non-executing)
**Overview:** A sticky note node with documentation, links, and broader system context (not part of the runtime).

**Nodes involved:**
- Sticky Note

#### Node: **Sticky Note**
- **Type / role:** `Sticky Note` — canvas annotation only.
- **Content preserved (includes link):**
  - “Email Outreach Automation”
  - YouTube reference: `@[youtube](1o4oREHQgLw)`
  - Notion documentation link: https://freemezie.notion.site/Email-Outreach-Automation-2d4c8a2a63e080dfa4e8e0fe6ea27894?source=copy_link
  - Mentions APIs: Wiza, OpenAI, Perplexity, Gmail
- **Edge cases:** None (non-executing)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Entry point; collect lead + enrichment intent | — | Switch | ## Email Outreach Automation / @[youtube](1o4oREHQgLw) / Notion: https://freemezie.notion.site/Email-Outreach-Automation-2d4c8a2a63e080dfa4e8e0fe6ea27894?source=copy_link / APIs: Wiza, OpenAI, Perplexity, Gmail |
| Switch | Switch | Route by enrichment type | On form submission | Email Finder; Phone Finder; Linkedin Finder | (same as above) |
| Email Finder | Wiza | Enrich email | Switch | Merge | (same as above) |
| Phone Finder | Wiza | Enrich phone | Switch | Merge | (same as above) |
| Linkedin Finder | Wiza | Find LinkedIn profile | Switch | Merge | (same as above) |
| Merge | Merge | Combine up to 3 enrichment results | Email Finder; Phone Finder; Linkedin Finder | Switch1 | (same as above) |
| Switch1 | Switch | Route based on status (failed/finished) | Merge | Failed To Find Enrichment; Clean Up | (same as above) |
| Failed To Find Enrichment | Form (completion) | Failure completion page | Switch1 | — | (same as above) |
| Clean Up | Code | Normalize merged enrichment into one lead object | Switch1 | Create or update a contact; Append or update row in sheet; Insert row | (same as above) |
| Create or update a contact | HubSpot | Upsert contact in HubSpot | Clean Up | — | (same as above) |
| Append or update row in sheet | Google Sheets | Upsert row into spreadsheet by email | Clean Up | Success Message Form | (same as above) |
| Insert row | Data Table | Store lead in n8n Data Table | Clean Up | — | (same as above) |
| Success Message Form | Form (completion) | Success completion page | Append or update row in sheet | — | (same as above) |
| Sticky Note | Sticky Note | Canvas annotation/documentation | — | — | ## Email Outreach Automation / @[youtube](1o4oREHQgLw) / Notion: https://freemezie.notion.site/Email-Outreach-Automation-2d4c8a2a63e080dfa4e8e0fe6ea27894?source=copy_link / APIs: Wiza, OpenAI, Perplexity, Gmail |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it (e.g.) **“Full Contact Enrichment Workflow”**.
   - Ensure workflow settings execution order is the default (this one uses `executionOrder: v1`).

2) **Add the trigger: Form Trigger**
   - Node: **Form Trigger** → rename to **“On form submission”**
   - Form title: **Linkedin**
   - Add fields:
     - Dropdown (required): **What Are You Searching For?**
       - Options: Find Email, Find Phone, Find Linkedin, Find All
     - Text fields (optional): Full Name, Company Name, Email, Phone, Linkedin
   - This node is the **entry point**.

3) **Add routing: Switch**
   - Node: **Switch** → rename to **“Switch”**
   - Create rules:
     - If `{{$json["What Are You Searching For?"]}}` equals `Find Email` → output “Find Email”
     - equals `Find Phone` → output “Find Phone”
     - equals `Find Linkedin` → output “Find Linkedin”
     - equals `Find All` → output “Find All”
   - Connect **On form submission → Switch**.

4) **Add Wiza enrichment nodes (3 nodes)**
   - Add node: **Wiza** → rename to **Email Finder**
     - Operation: Email finder (default)
     - Credentials: create/select **Wiza API** credentials
   - Add node: **Wiza** → rename to **Phone Finder**
     - Operation: **phoneFinder**
     - Credentials: same Wiza account
   - Add node: **Wiza** → rename to **Linkedin Finder**
     - Operation: **linkedinFinder**
     - Credentials: same Wiza account
   - Connect:
     - Switch (Find Email) → Email Finder
     - Switch (Find Phone) → Phone Finder
     - Switch (Find Linkedin) → Linkedin Finder
     - Switch (Find All) → Email Finder AND Phone Finder AND Linkedin Finder

5) **Add Merge**
   - Node: **Merge** → rename to **“Merge”**
   - Set **Number of inputs = 3**
   - Connect:
     - Email Finder → Merge (Input 1 / index 0)
     - Phone Finder → Merge (Input 2 / index 1)
     - Linkedin Finder → Merge (Input 3 / index 2)
   - Verify Merge mode so it emits output when the selected branches run (important when not all three branches execute).

6) **Add status Switch**
   - Node: **Switch** → rename to **“Switch1”**
   - Rules:
     - `{{$json.status}}` equals `failed` → output “Failed”
     - `{{$json.status}}` equals `finished` → output “Finished”
   - Connect **Merge → Switch1**.

7) **Add failure completion form**
   - Node: **Form** (operation: Completion) → rename to **“Failed To Find Enrichment”**
   - Completion title: **Failed To Find Enrichment Data**
   - Message: **The Wiza Back-End Could Not Find any Enrichment Data from the provided information**
   - Connect **Switch1 (Failed) → Failed To Find Enrichment**.

8) **Add normalization Code node**
   - Node: **Code** → rename to **“Clean Up”**
   - Paste logic equivalent to:
     - Combine all incoming items
     - Pick base fields from first item
     - Extract best email from `email` or `emails[0].email`
     - Extract best phone from `phone_number` or `phones[0].number`
     - Output one item with:
       `first_name,last_name,email,phone_number,company_name,website,location,job_title,linkedin_profile,company_url`
   - Connect **Switch1 (Finished) → Clean Up**.

9) **Add HubSpot upsert**
   - Node: **HubSpot** → rename to **“Create or update a contact”**
   - Authentication: **App Token**
   - Credentials: create/select HubSpot App Token credential
   - Email field: `{{$json.email}}`
   - Map additional fields:
     - firstName `{{$json.first_name}}`
     - lastName `{{$json.last_name}}`
     - phoneNumber `{{$json.phone_number}}`
     - jobTitle `{{$json.job_title}}`
     - city `{{$json.location}}`
     - companyName `{{$json.company_name}}`
     - linkedinUrl `{{$json.linkedin_profile}}`
   - Connect **Clean Up → Create or update a contact**.

10) **Add Google Sheets append-or-update**
   - Node: **Google Sheets** → rename to **“Append or update row in sheet”**
   - Credentials: Google Sheets OAuth2
   - Operation: **Append or Update**
   - Select Spreadsheet and Sheet tab
   - Set **Matching columns: email**
   - Mapping: auto-map to columns:
     - first_name, last_name, email, phone_number, company_name, website, location, job_title, linkedin_profile, company_url
   - Connect **Clean Up → Append or update row in sheet**.

11) **Add n8n Data Table insert**
   - Node: **Data Table** → rename to **“Insert row”**
   - Select/create Data Table **Leads**
   - Configure field mappings (recommended explicitly), e.g.:
     - email ← `{{$json.email}}`
     - name ← `{{$json.first_name}} {{$json.last_name}}`
     - title ← `{{$json.job_title}}`
     - location ← `{{$json.location}}`
     - linkedin_profile_url ← `{{$json.linkedin_profile}}`
     - phone ← `{{$json.phone_number}}`
     - company_name ← `{{$json.company_name}}`
     - company_domain ← `{{$json.website}}` (or company_url)
   - Connect **Clean Up → Insert row**.

12) **Add success completion form**
   - Node: **Form** (operation: Completion) → rename to **“Success Message Form”**
   - Title: **Success!**
   - Message: **Your new lead has been successfully added to your CRM**
   - Connect **Append or update row in sheet → Success Message Form**.
   - (This matches the JSON behavior where success depends on Sheets completing.)

13) **(Optional) Add a Sticky Note**
   - Add a sticky note with:
     - YouTube: `@[youtube](1o4oREHQgLw)`
     - Notion link: https://freemezie.notion.site/Email-Outreach-Automation-2d4c8a2a63e080dfa4e8e0fe6ea27894?source=copy_link

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Email Outreach Automation” + YouTube reference `@[youtube](1o4oREHQgLw)` | Sticky note on the canvas |
| Full Notion Documentation link (includes resources/templates) | https://freemezie.notion.site/Email-Outreach-Automation-2d4c8a2a63e080dfa4e8e0fe6ea27894?source=copy_link |
| Sticky note describes a broader system (AI research + email writing + Gmail drafts) not present in this specific JSON | The current workflow focuses on Wiza enrichment + syncing; OpenAI/Perplexity/Gmail are mentioned but not implemented as nodes here. |