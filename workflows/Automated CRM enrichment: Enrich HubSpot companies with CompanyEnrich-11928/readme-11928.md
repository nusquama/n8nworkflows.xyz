Automated CRM enrichment: Enrich HubSpot companies with CompanyEnrich

https://n8nworkflows.xyz/workflows/automated-crm-enrichment--enrich-hubspot-companies-with-companyenrich-11928


# Automated CRM enrichment: Enrich HubSpot companies with CompanyEnrich

## 1. Workflow Overview

**Purpose:** This workflow periodically finds recently created/updated **HubSpot companies**, extracts each company’s **domain**, calls the **CompanyEnrich** enrichment API, and updates the HubSpot company record with enriched attributes (location, socials, description, etc.). If enrichment fails, it routes the item to a logging/no-op path and continues processing the remaining companies.

**Target use cases:**
- Automated CRM data enrichment for sales/marketing ops
- Keeping company records up to date after new company creation or changes
- Bulk enrichment on a rolling window (e.g., last 7 days)

### 1.1 Scheduling & Company Selection (HubSpot)
Runs weekly (configurable) and fetches companies updated/created since a computed date.

### 1.2 Domain Extraction & Batch Loop Control
Normalizes HubSpot company payloads to `{ companyId, domain }` and processes companies in a loop using batching.

### 1.3 Enrichment Call (CompanyEnrich)
Calls CompanyEnrich “enrich by domain” endpoint for each domain.

### 1.4 Response Merge, Success Check, and HubSpot Update
Merges the enrichment response with the original `{companyId, domain}`, checks if the HTTP call succeeded, then updates the HubSpot company fields. Failures go to an “error log” branch and the loop continues.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & retrieving recently changed HubSpot companies
**Overview:** Triggers on a schedule, then queries HubSpot for companies created/updated within a configurable time window (default: last 7 days).  
**Nodes involved:** `Schedule Trigger`, `Get recently created/updated companies`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entry point; periodic execution.
- **Configuration:** Runs at an interval defined as **weeks** (weekly). (Exact cadence can be changed in node settings.)
- **Inputs/outputs:** No inputs (trigger). Outputs to HubSpot query node.
- **Edge cases / failures:**
  - Workflow won’t run if inactive (`active: false` in the workflow export).
  - Schedule misconfiguration can cause too-frequent runs (API rate limits).

#### Node: Get recently created/updated companies
- **Type / role:** `n8n-nodes-base.hubspot` — fetch HubSpot companies updated/created since a date.
- **Operation:** `company.getRecentlyCreatedUpdated`, `returnAll: true`
- **Authentication:** HubSpot **App Token** (`hubspotAppToken Credential`)
- **Key expression:**
  - `since` is computed as:  
    `new Date(new Date($json.timestamp).getTime() - 7*24*60*60*1000).toISOString().slice(0,10)`
  - This subtracts **7 days** from the trigger timestamp, formats to ISO, and keeps only `YYYY-MM-DD`.
- **Inputs/outputs:** Receives trigger item; outputs a list of companies to `Extract Domain`.
- **Edge cases / failures:**
  - **Auth/scopes:** requires HubSpot private app token with **company read** at minimum.
  - **Date format:** slicing to `YYYY-MM-DD` may or may not match the exact expected format for this HubSpot endpoint depending on n8n node implementation; if HubSpot expects a timestamp, results may be unexpected (too many/too few).
  - **Large datasets:** `returnAll: true` can be heavy; may hit rate limits or long executions.

---

### Block 2 — Domain extraction & loop/batch mechanics
**Overview:** Converts each HubSpot company into a minimal object with `companyId` and `domain`, then iterates through the list using `Split in Batches` to process items sequentially/iteratively.  
**Nodes involved:** `Extract Domain`, `Loop Over Items`

#### Node: Extract Domain
- **Type / role:** `n8n-nodes-base.function` — transforms HubSpot company payload into `{companyId, domain}`.
- **Configuration choices (interpreted):**
  - Attempts multiple possible field locations to find domain/website:
    - `src.properties.domain.value`, `src.properties.domain`,
    - `src.properties.website.value`, `src.properties.website`,
    - `src.domain`, `src.website`,
    - fallback `""`
  - Company ID is found from: `src.id`, `src.objectId`, `src["objectId"]`, `src.companyId`, fallback `""`
- **Key variables produced:** `companyId`, `domain`
- **Inputs/outputs:** Input is each HubSpot company item; output goes to `Loop Over Items`.
- **Edge cases / failures:**
  - Missing/empty `domain` will still pass through (domain becomes `""`), which will likely cause enrichment failure or useless results.
  - If HubSpot node returns a different shape (e.g., `properties` structure differs), extraction may fail silently (domain empty).

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — controls iterative processing.
- **Configuration:** `reset: false` (keeps state within the execution), batch size not explicitly set in the provided JSON (n8n default applies).
- **Connections (important):**
  - **Input:** from `Extract Domain` and also from `HubSpot - Update Company` and `On Error (Log)` (to continue looping).
  - **Output index 1:** sends current batch item to both `CompanyEnrich - Enrich by Domain` and `Merge`.
  - **Output index 0:** unused (empty connection) in this workflow.
- **Edge cases / failures:**
  - If batch size is large, parallel downstream calls may increase rate limiting risk (though the structure suggests a loop, the dual connection from output index 1 can still process per batch item).
  - Miswiring outputs (index 0 vs 1) can cause the loop never to start.

---

### Block 3 — CompanyEnrich enrichment call
**Overview:** Calls CompanyEnrich API with the extracted domain and returns enriched company data.  
**Nodes involved:** `CompanyEnrich - Enrich by Domain`

#### Node: CompanyEnrich - Enrich by Domain
- **Type / role:** `n8n-nodes-base.httpRequest` — external API call for enrichment.
- **Request configuration:**
  - **Method:** (not explicitly shown; n8n HTTP Request defaults to GET unless configured otherwise; this node uses query parameters and no body, so it behaves like GET.)
  - **URL:** `https://api.companyenrich.com/companies/enrich`
  - **Query parameter:** `domain = {{ $json.domain }}`
  - **Headers (JSON):**
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Accept: application/json`
  - `sendQuery: true`, `sendHeaders: true`, `specifyHeaders: json`
- **Inputs/outputs:** Input from `Loop Over Items` (current item). Output goes to `Merge` input index 1.
- **Edge cases / failures:**
  - **Auth:** placeholder token must be replaced; invalid token → 401/403.
  - **Empty domain:** likely yields 4xx error or empty result.
  - **HTTP errors:** depending on node settings, n8n may either throw and stop execution or return an error object; this workflow’s later “statusCode isEmpty” check suggests it expects a successful JSON payload without `statusCode`.
  - **Rate limits/timeouts:** enrichment API could throttle or time out.

---

### Block 4 — Merge + success routing + HubSpot update (and error path)
**Overview:** Merges the original `{companyId, domain}` with the enrichment result, checks if the enrichment response indicates success, and updates HubSpot fields; otherwise routes to a no-op “log” node and continues the loop.  
**Nodes involved:** `Merge`, `Check Enrich Success`, `HubSpot - Update Company`, `On Error (Log)`

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines data streams.
- **Configuration:** `mode: combine`, `combineBy: combineByPosition`
  - Input 0: original item from `Loop Over Items`
  - Input 1: enrichment response from `CompanyEnrich - Enrich by Domain`
  - Combines items by their positional index.
- **Inputs/outputs:** Output to `Check Enrich Success`.
- **Edge cases / failures:**
  - If one branch produces fewer/more items than the other, positional combine can misalign data.
  - If HTTP Request returns an error item structure, merge still happens but downstream mapping may break.

#### Node: Check Enrich Success
- **Type / role:** `n8n-nodes-base.if` — routes success vs failure.
- **Condition logic (interpreted):**
  - Checks whether `String($json.statusCode)` **is empty**:
    - Expression: `{{ $json && $json.statusCode ? String($json.statusCode) : '' }}`
    - If empty → treated as “success” path (output 0)
    - If not empty → treated as “error” path (output 1)
- **Inputs/outputs:**
  - Input from `Merge`
  - Output 0 → `HubSpot - Update Company`
  - Output 1 → `On Error (Log)`
- **Edge cases / failures:**
  - This success heuristic assumes **successful responses do not contain `statusCode`** and failures do. That depends on how the HTTP Request node is configured and how n8n represents errors.
  - If CompanyEnrich returns a JSON payload that legitimately includes a `statusCode` field, it may be misclassified as failure.

#### Node: HubSpot - Update Company
- **Type / role:** `n8n-nodes-base.hubspot` — writes enriched fields into HubSpot company record.
- **Operation:** `company.update`
- **Authentication:** HubSpot **App Token** (`hubspotAppToken Credential`)
- **Key fields mapped (examples):**
  - `companyId: {{ $json.companyId }}`
  - `name: {{ $json.name }}`
  - `companyDomainName: {{ $json.domain }}`
  - `websiteUrl: {{ $json.website }}`
  - `description: {{ $json.description }}`
  - Address/location:
    - `streetAddress: {{ $json.location.address }}`
    - `city: {{ $json.location.city.name }}`
    - `stateRegion: {{ $json.location.state.name }}`
    - `countryRegion: {{ $json.location.country.name }}`
    - `postalCode: {{ $json.location.postal_code }}`
    - `phoneNumber: {{ $json.location.phone }}`
  - Socials:
    - `twitterHandle: {{ $json.socials.twitter_url }}`
    - `linkedInCompanyPage: {{ $json.socials.linkedin_url }}`
  - `yearFounded: {{ $json.founded_year }}`
- **Inputs/outputs:** Input from `Check Enrich Success` success branch. Output loops back into `Loop Over Items` to continue.
- **Edge cases / failures:**
  - **Auth/scopes:** requires HubSpot private app token with **company write**.
  - **Property types:** HubSpot expects certain property formats; sending objects/arrays or invalid values can fail the update.
  - **Missing nested fields:** if `location` or `socials` is missing, expressions may evaluate to `undefined` and HubSpot may reject or ignore fields depending on API behavior.
  - **Mapping constraint (from sticky note):** sending multiple values to a single HubSpot property can error (e.g., if expression produces array).

#### Node: On Error (Log)
- **Type / role:** `n8n-nodes-base.noOp` — placeholder for error handling; currently just passes data through.
- **Configuration:** No operation (does not log by itself; it simply continues the workflow).
- **Inputs/outputs:** Input from `Check Enrich Success` failure branch; output back to `Loop Over Items`.
- **Edge cases / failures:**
  - Since it doesn’t record anything, failures may be invisible unless execution logs are inspected. Consider replacing/augmenting with Slack/Email/Datastore logging.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Start workflow on a weekly cadence | — | Get recently created/updated companies | ## Schedule Trigger & Extract Domain  \nControl how far back companies are enriched:  \n1. **Schedule Trigger:**  \n   Set how often the workflow runs.  \n2. **HubSpot Get Companies node:**  \n   Change the number in `7*24*60*60*1000`.  \n   - `7` = last 7 days  \n   -  Change this number to any value you want (e.g. `1` = last 1 day, `14` = last 14 days) |
| Get recently created/updated companies | HubSpot | Fetch recently created/updated companies | Schedule Trigger | Extract Domain | ## Schedule Trigger & Extract Domain  \nControl how far back companies are enriched:  \n1. **Schedule Trigger:**  \n   Set how often the workflow runs.  \n2. **HubSpot Get Companies node:**  \n   Change the number in `7*24*60*60*1000`.  \n   - `7` = last 7 days  \n   -  Change this number to any value you want (e.g. `1` = last 1 day, `14` = last 14 days) |
| Extract Domain | Function | Normalize items to `{companyId, domain}` | Get recently created/updated companies | Loop Over Items | ## Schedule Trigger & Extract Domain  \nControl how far back companies are enriched:  \n1. **Schedule Trigger:**  \n   Set how often the workflow runs.  \n2. **HubSpot Get Companies node:**  \n   Change the number in `7*24*60*60*1000`.  \n   - `7` = last 7 days  \n   -  Change this number to any value you want (e.g. `1` = last 1 day, `14` = last 14 days) |
| Loop Over Items | Split In Batches | Iterate through companies in batches and keep looping | Extract Domain; HubSpot - Update Company; On Error (Log) | CompanyEnrich - Enrich by Domain; Merge | ## Enrichment Loop  \nEnter your CompanyEnrich API key in the **HTTP Request** node where it says `YOUR_API_KEY`. |
| CompanyEnrich - Enrich by Domain | HTTP Request | Call CompanyEnrich API using domain | Loop Over Items | Merge | ## Enrichment Loop  \nEnter your CompanyEnrich API key in the **HTTP Request** node where it says `YOUR_API_KEY`. |
| Merge | Merge | Combine original item + enrichment response | Loop Over Items; CompanyEnrich - Enrich by Domain | Check Enrich Success |  |
| Check Enrich Success | IF | Route based on whether response indicates an error | Merge | HubSpot - Update Company; On Error (Log) |  |
| HubSpot - Update Company | HubSpot | Update company properties with enriched data | Check Enrich Success | Loop Over Items | ## Update Companies  \nControl which properties are sent to HubSpot by mapping them in the **HubSpot Update Company** node.  \n⚠️ Be careful to map **only one value per field** to avoid HubSpot API errors. |
| On Error (Log) | No Operation | Placeholder error path; continues loop | Check Enrich Success | Loop Over Items |  |
| Sticky Note | Sticky Note | Documentation / instructions | — | — | ## AI COMPANY ENRICHMENT FOR NEW OR UPDATED HUBSPOT COMMPANIES  \n### 📌 How This Workflow Works?  \n1. **Schedule Trigger:** Finds the newly created or updated companies in a set time period.  \n2. **Extract Domain:** Extracts the domain seeds of the companies for CompanyEnrich enrichment API.  \n3. **Hubspot Update Company:** Updates the corresponding company with enriched data if enrichment success is false we log the error  \n### 🛠️ Before using  \n✔ Create an Hubspot private app with company read and write scopes, add your app's credentials for both hubspot nodes.    \n✔ Ensure companies have a valid domain (required for enrichment)    \n✔ Create an Company Enrich account and enter your api key to HTTP Request node's header where it says `YOUR_API_KEY`.    \n✔ Map the enriched datas in Hubspot Company Update nodes.  \n### CUSTOMIZATION  \n1. **Filters:** Change the time period in schedule trigger and get recently updated companies nodes    \n2. **Fields to Update:** Map the custom company properties in Update companies node  \nFor more customization instructions, check the respective node’s sticky notes. |
| Sticky Note1 | Sticky Note | Documentation / schedule & lookback | — | — | ## Schedule Trigger & Extract Domain  \nControl how far back companies are enriched:  \n1. **Schedule Trigger:**  \n   Set how often the workflow runs.  \n2. **HubSpot Get Companies node:**  \n   Change the number in `7*24*60*60*1000`.  \n   - `7` = last 7 days  \n   -  Change this number to any value you want (e.g. `1` = last 1 day, `14` = last 14 days) |
| Sticky Note2 | Sticky Note | Documentation / update mapping caution | — | — | ## Update Companies  \nControl which properties are sent to HubSpot by mapping them in the **HubSpot Update Company** node.  \n⚠️ Be careful to map **only one value per field** to avoid HubSpot API errors. |
| Sticky Note3 | Sticky Note | Documentation / API key placement | — | — | ## Enrichment Loop  \nEnter your CompanyEnrich API key in the **HTTP Request** node where it says `YOUR_API_KEY`. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: `Enrich-Hubspot-Companies`.

2. **Add “Schedule Trigger”** (`Schedule Trigger` node)
   - Set interval to **Weeks** (weekly).  
   - Connect to the HubSpot fetch node.

3. **Add “HubSpot” node** named `Get recently created/updated companies`
   - **Resource:** Company  
   - **Operation:** Get Recently Created/Updated  
   - Enable **Return All**
   - **Additional field** `since` (expression):
     - Use:  
       `{{ new Date(new Date($json.timestamp).getTime() - 7*24*60*60*1000).toISOString().slice(0,10) }}`
   - **Credentials:** Create/use HubSpot **Private App Token** credential (App Token).
     - Ensure scopes include at least **crm.objects.companies.read**.
   - Connect to `Extract Domain`.

4. **Add “Function” node** named `Extract Domain`
   - Paste logic that outputs:
     - `companyId` from `id/objectId/companyId`
     - `domain` from `properties.domain / properties.website / domain / website`
   - Output item shape must be:
     - `{ json: { companyId, domain } }`
   - Connect to `Loop Over Items`.

5. **Add “Split In Batches”** node named `Loop Over Items`
   - Keep `reset` = false (default shown).
   - Connect **Output 1** (second output) to:
     - `CompanyEnrich - Enrich by Domain`
     - `Merge` (input 0)
   - (Output 0 remains unused, matching the provided workflow.)

6. **Add “HTTP Request”** node named `CompanyEnrich - Enrich by Domain`
   - **URL:** `https://api.companyenrich.com/companies/enrich`
   - Enable **Send Query Parameters**
     - Add `domain` = `{{ $json.domain }}`
   - Enable **Send Headers** and set headers as JSON:
     - `Authorization: Bearer <YOUR_COMPANYENRICH_TOKEN>`
     - `Accept: application/json`
   - Connect to `Merge` (input 1).

7. **Add “Merge”** node named `Merge`
   - **Mode:** Combine
   - **Combine By:** Position
   - Inputs:
     - Input 0 from `Loop Over Items`
     - Input 1 from `CompanyEnrich - Enrich by Domain`
   - Output to `Check Enrich Success`.

8. **Add “IF”** node named `Check Enrich Success`
   - Condition: String → **isEmpty**
     - Value: `{{ $json && $json.statusCode ? String($json.statusCode) : '' }}`
   - Output 0 (true) → `HubSpot - Update Company`
   - Output 1 (false) → `On Error (Log)`

9. **Add “HubSpot” node** named `HubSpot - Update Company`
   - **Resource:** Company
   - **Operation:** Update
   - **Company ID:** `{{ $json.companyId }}`
   - **Update Fields:** map from the enrichment response, e.g.:
     - Name, Domain, Website, Description
     - Location fields (address/city/state/country/postal/phone)
     - Social fields (Twitter/LinkedIn URLs)
     - Founded year
   - **Credentials:** same HubSpot private app token credential
     - Ensure scopes include **crm.objects.companies.write**
   - Connect output back to `Loop Over Items` (to continue the loop).

10. **Add “No Operation”** node named `On Error (Log)`
   - Leave default.
   - Connect output back to `Loop Over Items` (to continue despite errors).
   - (Optional but recommended) Replace/extend with a real logging node (e.g., Slack, email, database, or “Add to execution data” pattern).

11. **Add Sticky Notes (optional but matches original)**
   - Add notes describing:
     - Schedule + lookback window (`7*24*60*60*1000`)
     - Where to place CompanyEnrich API key
     - Warning about mapping only one value per HubSpot field

12. **Activate the workflow**
   - Ensure credentials are set.
   - Run a manual execution once to validate:
     - domains are extracted
     - CompanyEnrich response schema matches mappings
     - HubSpot updates succeed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a HubSpot private app with company read/write scopes and use its app token credential in both HubSpot nodes. | Credential prerequisite from sticky notes |
| Ensure companies have a valid domain; enrichment depends on it. | Data prerequisite |
| Add CompanyEnrich API key/token in HTTP Request Authorization header (`Bearer ...`). | External API prerequisite |
| Customize enrichment window by changing `7*24*60*60*1000` multiplier in the HubSpot “since” expression. | Filtering customization |
| Map enriched data carefully; send only one value per HubSpot field to avoid API errors. | HubSpot update constraint |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.