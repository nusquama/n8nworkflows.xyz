Detect job and company changes with Lusha bulk enrichment and HubSpot

https://n8nworkflows.xyz/workflows/detect-job-and-company-changes-with-lusha-bulk-enrichment-and-hubspot-13351


# Detect job and company changes with Lusha bulk enrichment and HubSpot

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Detect job and company changes with Lusha bulk enrichment and HubSpot  
**Workflow name (in JSON):** Contact Job Change & Promotion Enrichment with Lusha

**Purpose:**  
This workflow runs on a daily schedule, pulls a batch of contacts from HubSpot, re-enriches them in bulk using Lusha, compares the enriched data against the existing CRM fields (job title and company), and—when changes are detected—updates HubSpot and notifies a Slack channel.

**Target use cases:**
- Account Executives (AEs) and Customer Success Managers (CSMs) tracking “champions” and key stakeholders.
- Detecting job changes/promotions to trigger re-engagement or account mapping updates.
- Keeping CRM data fresh automatically (company/job title/phone).

### 1.1 Logical Blocks
1. **1.1 Scheduling & CRM Contact Pull**  
   Daily trigger → fetch HubSpot contacts (limited batch).
2. **1.2 Payload Preparation for Bulk Enrichment**  
   Format contacts (emails) into Lusha bulk payload (single request).
3. **1.3 Bulk Enrichment via Lusha**  
   Call Lusha Bulk Enrichment API via community node.
4. **1.4 Change Detection & Routing**  
   Compare Lusha results with HubSpot values → IF “changed”.
5. **1.5 CRM Update & Slack Alert**  
   Update HubSpot contact fields + notify Slack.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Scheduling & CRM Contact Pull
**Overview:** Runs daily and retrieves a defined number of HubSpot contacts to check for job/company changes.

**Nodes involved:**
- **Daily Schedule** (Schedule Trigger)
- **Get CRM Contacts** (HubSpot)

#### Node: Daily Schedule
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entry point, periodic execution.
- **Configuration choices:** Runs every **24 hours** (`hoursInterval: 24`).
- **Inputs/Outputs:** No inputs; outputs into **Get CRM Contacts**.
- **Failure/edge cases:**
  - Missed executions if n8n instance is down.
  - Time zone considerations depend on n8n instance settings.

#### Node: Get CRM Contacts
- **Type / role:** `n8n-nodes-base.hubspot` — fetches contacts from HubSpot.
- **Operation:** `contact → getAll`
- **Configuration choices:**
  - `returnAll: false`, `limit: 100` (only first 100 returned).
  - `formSubmissionMode: none` (avoids including form submissions payload).
- **Credentials:** HubSpot OAuth2 required.
- **Inputs/Outputs:**
  - Input: from **Daily Schedule**
  - Output: to **Format for Bulk Enrichment**
- **Failure/edge cases:**
  - OAuth token expiry / missing scopes.
  - Pagination is not handled because `returnAll` is false; only first page is processed (risk: missing most contacts).
  - HubSpot API rate limits (429) on frequent pulls or larger limits.

---

### Block 1.2 — Payload Preparation for Bulk Enrichment
**Overview:** Converts HubSpot contact list into the Lusha bulk enrichment JSON payload, using only contacts that have an email.

**Nodes involved:**
- **Format for Bulk Enrichment** (Code)

#### Node: Format for Bulk Enrichment
- **Type / role:** `n8n-nodes-base.code` — transforms HubSpot records into Lusha payload.
- **Key logic/config:**
  - Reads **all incoming HubSpot items**: `const items = $input.all();`
  - Filters out contacts without email: `filter(item => item.json.properties?.email)`
  - Creates `contacts` array with:
    - `contactId`: sequential string counter `"1"`, `"2"`, …
    - `email`: from HubSpot `properties.email`
  - Wraps into payload: `{ contacts, metadata: {} }`
  - Outputs a single item with `contactsPayload` **stringified JSON**:
    - `return [{ json: { contactsPayload: JSON.stringify(payload) } }];`
- **Inputs/Outputs:**
  - Input: from **Get CRM Contacts**
  - Output: to **Re-Enrich All in Bulk**
- **Failure/edge cases:**
  - If **no contacts have email**, `contacts` becomes empty; Lusha may reject or return empty results.
  - Stringified JSON must match what the Lusha node expects (here it’s passed to `contactsPayloadJson`).
  - The `contactId` generated here is **not** the HubSpot ID; it’s just a sequential identifier for Lusha payload tracking.

---

### Block 1.3 — Bulk Enrichment via Lusha
**Overview:** Sends a single bulk request to Lusha to re-enrich all emails and return updated job/company and contact data.

**Nodes involved:**
- **Re-Enrich All in Bulk** (Lusha community node)

#### Node: Re-Enrich All in Bulk
- **Type / role:** `@lusha-org/n8n-nodes-lusha.lusha` — Lusha bulk enrichment integration.
- **Operation:** `enrichBulk`
- **Configuration choices:**
  - `bulkType: json`
  - `contactsPayloadJson: ={{ $json.contactsPayload }}`
  - `contactBulkAdditionalOptions: {}` (none set)
- **Credentials:** Lusha API credential required.
- **Inputs/Outputs:**
  - Input: from **Format for Bulk Enrichment**
  - Output: to **Detect Changes**
- **Failure/edge cases:**
  - Missing/invalid Lusha API key, plan restrictions, quota exceeded.
  - Payload format mismatch (string vs object) depending on node implementation/version.
  - Large payload sizes may hit request limits/timeouts.

---

### Block 1.4 — Change Detection & Routing
**Overview:** Compares enriched job title/company from Lusha to the stored HubSpot fields and flags contacts with differences.

**Nodes involved:**
- **Detect Changes** (Code)
- **Has Job Changes?** (IF)

#### Node: Detect Changes
- **Type / role:** `n8n-nodes-base.code` — data reconciliation logic.
- **Key expressions/variables:**
  - Pulls Lusha results: `const items = $input.all();`
  - Pulls original HubSpot contacts directly by node name: `const crmContacts = $('Get CRM Contacts').all();`
  - Builds lookup: `crmByEmail[email] = c.json;` (email lowercased)
  - For each Lusha record:
    - `email = (lusha.primaryEmail || '').toLowerCase()`
    - Compares:
      - HubSpot `properties.jobtitle` vs Lusha `jobTitle`
      - HubSpot `properties.company` vs Lusha `companyName`
    - Only records changes if **both current and new values are non-empty**:
      - `if (newCompany && currentCompany && newCompany !== currentCompany) ...`
      - `if (newTitle && currentTitle && newTitle !== currentTitle) ...`
  - Output fields per contact:
    - `contactId: crm.id || ''` (HubSpot internal ID)
    - `hasChanges: changes.length > 0`
    - `changes: [{ field, from, to }, ...]`
    - `newJobTitle`, `newCompany`, `newSeniority`, `newPhone`, `checkedAt`
- **Inputs/Outputs:**
  - Input: from **Re-Enrich All in Bulk**
  - Output: to **Has Job Changes?**
- **Failure/edge cases:**
  - **Matching by email**: if Lusha returns a different email field or none (`primaryEmail` empty), contact won’t match HubSpot; `contactId` becomes empty and downstream update will fail.
  - If HubSpot has empty job title/company, changes are not flagged (by design). This can miss “newly filled” data.
  - Multiple HubSpot contacts sharing same email: last one wins in `crmByEmail`.
  - Node-name dependency: `$('Get CRM Contacts')` will break if the node is renamed.

#### Node: Has Job Changes?
- **Type / role:** `n8n-nodes-base.if` — routes only changed contacts.
- **Condition:** `{{ $json.hasChanges }}` is `true`.
- **Inputs/Outputs:**
  - Input: from **Detect Changes**
  - True output: to **Update CRM Record** and **Alert Rep on Slack**
  - False output: unused (terminates)
- **Failure/edge cases:**
  - If `hasChanges` is missing/non-boolean, condition may behave unexpectedly.
  - No “false path” handling (e.g., logging/metrics) is implemented.

---

### Block 1.5 — CRM Update & Slack Alert
**Overview:** For contacts flagged as changed, updates HubSpot fields and posts a Slack notification summarizing the change.

**Nodes involved:**
- **Update CRM Record** (HubSpot)
- **Alert Rep on Slack** (Slack)

#### Node: Update CRM Record
- **Type / role:** `n8n-nodes-base.hubspot` — updates the HubSpot contact record.
- **Operation:** `contact → update`
- **Key mappings (updateFields):**
  - `contactId: {{ $json.contactId }}`
  - `phone: {{ $json.newPhone }}`
  - `company: {{ $json.newCompany }}`
  - `jobtitle: {{ $json.newJobTitle }}`
- **Credentials:** HubSpot OAuth2 required.
- **Inputs/Outputs:**
  - Input: from **Has Job Changes?** (true branch)
  - Output: none connected (workflow ends after update for that item)
- **Failure/edge cases:**
  - If `contactId` is empty (no match), HubSpot update will fail.
  - Overwriting: this replaces HubSpot fields even if Lusha returns empty/null (depends on what Lusha returns; current logic does not prevent writing blanks).
  - Rate limiting when many changes occur at once.

#### Node: Alert Rep on Slack
- **Type / role:** `n8n-nodes-base.slack` — posts a message to Slack.
- **Configuration choices:**
  - Channel: `#job-changes`
  - Text uses Slack markdown + inline JS in expression:
    - `{{ $json.changes.map(c => \`• ${c.field}: "${c.from}" → "${c.to}"\`).join('\n') }}`
  - Includes: name, email, bullet list of changed fields, confirmation line.
- **Credentials:** Slack OAuth2 required.
- **Inputs/Outputs:**
  - Input: from **Has Job Changes?** (true branch)
  - Output: none connected
- **Failure/edge cases:**
  - Slack OAuth missing scopes (`chat:write`), or channel not found/private without bot membership.
  - Expression rendering issues if `changes` is not an array (should be, per code).
  - Message length limits if many fields are added later.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Job Change & Promotion Enrichment | Sticky Note | Documentation / context | — | — | ## Contact Job Change & Promotion Detection with Lusha … Configure Slack channel for alerts |
| 📅 1. Daily CRM Pull | Sticky Note | Documentation for Block 1 | — | — | A daily schedule trigger pulls your existing contacts from HubSpot for re-enrichment. Nodes: Schedule Trigger → HubSpot Get Contacts … |
| 🔄 2. Bulk Re-Enrich with Lusha | Sticky Note | Documentation for Block 2/3 | — | — | All contacts are formatted and sent to the Lusha Bulk Enrichment API … [Lusha API docs](https://www.lusha.com/docs/) |
| 📤 3. Detect Changes & Alert Reps | Sticky Note | Documentation for Block 4/5 | — | — | A code node compares fresh Lusha data … Nodes: Detect Changes → Has Changes? → Update CRM + Slack Alert … |
| Daily Schedule | Schedule Trigger | Daily entry trigger | — | Get CRM Contacts | A daily schedule trigger pulls your existing contacts from HubSpot for re-enrichment. Nodes: Schedule Trigger → HubSpot Get Contacts … |
| Get CRM Contacts | HubSpot | Pull contacts from HubSpot | Daily Schedule | Format for Bulk Enrichment | A daily schedule trigger pulls your existing contacts from HubSpot for re-enrichment. Nodes: Schedule Trigger → HubSpot Get Contacts … |
| Format for Bulk Enrichment | Code | Build Lusha bulk payload from HubSpot emails | Get CRM Contacts | Re-Enrich All in Bulk | All contacts are formatted and sent to the Lusha Bulk Enrichment API … |
| Re-Enrich All in Bulk | Lusha (community) | Bulk enrich contacts via Lusha | Format for Bulk Enrichment | Detect Changes | All contacts are formatted and sent to the Lusha Bulk Enrichment API … [Lusha API docs](https://www.lusha.com/docs/) |
| Detect Changes | Code | Compare Lusha vs HubSpot and flag differences | Re-Enrich All in Bulk | Has Job Changes? | A code node compares fresh Lusha data against stored CRM records … |
| Has Job Changes? | IF | Route only changed contacts | Detect Changes | Update CRM Record; Alert Rep on Slack | A code node compares fresh Lusha data … Nodes: Detect Changes → Has Changes? → Update CRM + Slack Alert … |
| Update CRM Record | HubSpot | Update HubSpot contact fields | Has Job Changes? (true) | — | A code node compares fresh Lusha data … Nodes: Detect Changes → Has Changes? → Update CRM + Slack Alert … |
| Alert Rep on Slack | Slack | Notify channel about job/company change | Has Job Changes? (true) | — | A code node compares fresh Lusha data … Nodes: Detect Changes → Has Changes? → Update CRM + Slack Alert … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n. Set name to **“Contact Job Change & Promotion Enrichment with Lusha”** (or your preferred name).

2. **Add Trigger: “Schedule Trigger”**
   - Node type: **Schedule Trigger**
   - Configure interval: **Every 24 hours** (or set daily at a specific time if preferred).
   - Name it: **Daily Schedule**

3. **Add HubSpot node: “Get CRM Contacts”**
   - Node type: **HubSpot**
   - Credentials: **HubSpot OAuth2**
     - Ensure scopes allow reading contacts.
   - Resource: **Contact**
   - Operation: **Get All**
   - Set **Return All = false**
   - Set **Limit = 100** (adjust based on volume and Lusha plan)
   - Connect: **Daily Schedule → Get CRM Contacts**

4. **Add Code node: “Format for Bulk Enrichment”**
   - Node type: **Code**
   - Paste logic to:
     - read all HubSpot items
     - filter contacts without `properties.email`
     - build payload: `{ contacts: [{contactId, email}], metadata: {} }`
     - output `contactsPayload` as **JSON string**
   - Use essentially this structure (adapt if needed):
     - Output one item: `{ contactsPayload: JSON.stringify(payload) }`
   - Connect: **Get CRM Contacts → Format for Bulk Enrichment**

5. **Install and add Lusha community node**
   - Install `@lusha-org/n8n-nodes-lusha` in your n8n environment (Community Nodes).
   - Add node type: **Lusha**
   - Name it: **Re-Enrich All in Bulk**
   - Credentials: **Lusha API**
   - Operation: **Bulk Enrichment** (enrichBulk)
   - Input type: **JSON**
   - Set `contactsPayloadJson` to expression: `{{ $json.contactsPayload }}`
   - Connect: **Format for Bulk Enrichment → Re-Enrich All in Bulk**

6. **Add Code node: “Detect Changes”**
   - Node type: **Code**
   - Implement comparison:
     - Pull Lusha items from input
     - Pull HubSpot items using node reference: `$('Get CRM Contacts').all()`
     - Build `crmByEmail` map
     - Compare `company` and `jobtitle` vs Lusha `companyName` and `jobTitle`
     - Output `hasChanges`, `changes[]`, `contactId`, and “new…” fields
   - Connect: **Re-Enrich All in Bulk → Detect Changes**

7. **Add IF node: “Has Job Changes?”**
   - Node type: **IF**
   - Condition: Boolean → `{{ $json.hasChanges }}` **is true**
   - Connect: **Detect Changes → Has Job Changes?**

8. **Add HubSpot node: “Update CRM Record”**
   - Node type: **HubSpot**
   - Credentials: **HubSpot OAuth2** (same as above)
   - Resource: **Contact**
   - Operation: **Update**
   - Contact ID: `{{ $json.contactId }}`
   - Update fields:
     - `company = {{ $json.newCompany }}`
     - `jobtitle = {{ $json.newJobTitle }}`
     - `phone = {{ $json.newPhone }}`
   - Connect from IF **true** output: **Has Job Changes? (true) → Update CRM Record**

9. **Add Slack node: “Alert Rep on Slack”**
   - Node type: **Slack**
   - Credentials: **Slack OAuth2**
     - Ensure `chat:write` scope and bot is in the target channel.
   - Channel: `#job-changes` (or your channel)
   - Message text: include name/email and list changes using an expression that joins `$json.changes`.
   - Connect from IF **true** output: **Has Job Changes? (true) → Alert Rep on Slack**

10. **(Optional but recommended) Add error handling**
   - Add an Error Trigger workflow or add branches/logging for:
     - missing `contactId`
     - Lusha returning empty `primaryEmail`
     - HubSpot update failures / Slack failures

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Lusha community node before using “Re-Enrich All in Bulk”. | Mentioned in workflow notes. |
| Adjust HubSpot contact limit to match Lusha API plan and contact volume. | Sticky note “📅 1. Daily CRM Pull”. |
| Lusha API documentation | https://www.lusha.com/docs/ |
| Slack alerts are configured to post in `#job-changes`; customize message/channel to your process. | Sticky note “📤 3. Detect Changes & Alert Reps”. |